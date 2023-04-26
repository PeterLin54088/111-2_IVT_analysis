# 111-2_IVT_analysis

## 資料製作

### 製作原始資料.nc檔 (dataset.nc)
<div>
  資料來源：ERA5 </br>
  首先，資料的空間是全球範圍，時間是1979-2021，變數是U、V、Q。原始的資料是依據變數類型、時間、高度等，分成了數百份.nc檔。</br>
  為了整合資料並選擇出適當的空間、時間和變數，使用了CDO(Climate Data Operators)對這數百份.nc檔做了以下幾件事：</br>
  <ol>
    <li>切割 60E ~ 105E || 0N ~ 25N</li>
    <li>增加 氣壓維度</li>
    <li>黏合 高度維度</li>
    <li>黏合 時間維度</li>
    <li>黏合 U、V、Q變數</li>
  </ol>
  至此，我製造出dataset.nc，其維度有"時間"、"氣壓"、"緯度"、"經度"，變數為"U"、"V"、"Q"。</br>
</div>

### 製作低頻循環.nc檔 (seasonal_cycle.nc)
<div>
  先給出一些假設</br>
  <ul>
    <li>原始資料 = 短周期波動 + 中週期波動 + 長周期波動</li>
    <li>長周期波動的振幅很小</li>
  </ul>
  並給出一些定義</br>
  <ul>
    <li>長周期波動：週期一年以上</li>
    <li>中周期波動：週期一年~週期三個月</li>
    <li>短周期波動：週期三個月以下</li>
  </ul>
  這個獨立研究想看的是季節轉換，而轉換的時間尺度相對短，所以我更在意短周期波動。因此，我需要移除掉多餘的，非短周期的波動。</br>
  考慮到assumption，這裡就不處理長周期波動，只打算移除中周期波動，為此，使用了CDO(Climate Data Operators)對上述的原始資料.nc檔做了以下幾件事：</br>
  <ol>
    <li>計算氣候日平均</li>
    <li>計算季節循環 (lowpass filter, f = 3)</li>
  </ol>
  至此，我透過dataset.nc得到seasonal_cycle.nc。</br>
</div>

## 資料前處理

### 計算IVT & 濾除低頻循環 (IVT_raw.npz)
<div>
  真正期望預報的資料類型為IVT(Integrated water Vapor Transport), 所以我必須計算出IVT，我應當得到兩個變數：原始的IVT 和 季節循環的IVT。</br>
  然後，將原始的IVT和季節循環的IVT相減，得到我期望的，主要由短周期波動所組成的filtered IVT。</br>
</div>

### 降低維度 (IVT_SVD.npz)
<div>
  即便我得到了IVT，但這個IVT的維度過大，於是這裡使用SVD(Singular Value Decomposition)將IVT分解為</br>
  <ul>
    <li>空間結構：EOF (Empirical Orthogonal Functions)</li>
    <li>時間結構：Amplitude time series</li>
    <li>特徵值：Singular value</li>
    <li>Reference：<a href="https://depts.washington.edu/ocean423/notes/EOF.notes.pdf">SVD Introduction</a></li>
  </ul>
  接著，我透過計算各個EOF的解釋方差比(explained variance ratio)，找出最重要(90%)的前數個EOF，在此我使用了前38個EOF。</br>
  最後，我使用上述得到的EOF對應的amplitude time series作為我真正實作預報的資料。</br>
</div>

### 資料分組 (DataSet and DataLoader) & 資料標準化 (*scaler.gz)
<div>
  為了建立ML-based的模式，需要將全部資料分類為三個組別，訓練組(train set)、驗證組(valid set)、測試組(test set)。</br>
  以下先決定各自的資料比例</br>
  <ul>
    <li>全部資料：16/16</li>
    <li>訓練組：12/16</li>
    <li>驗證組：1/16</li>
    <li>測試組：3/16</li>
  </ul>
  接著依照年分先後依序切分資料，最後得到各資料組別的大致對應年分</br>
  <ul>
    <li>全部資料：1979 ~ 2021</li>
    <li>訓練組：1979 ~ 2011</li>
    <li>驗證組：2011 ~ 2013</li>
    <li>測試組：2013 ~ 2021</li>
  </ul>
  因為ML-based的模式對資料的尺度相當敏感，需要在保留資料特徵的情況下，將資料映射到另一個更好的向量空間。</br>
  為此，我使用了MinMaxScaler，其公式如下</br>
  <ul>
    <li>X_std = (X - X.min(axis=0)) / (X.max(axis=0) - X.min(axis=0))</li>
    <li>X_scaled = X_std * (max - min) + min</li>
    <li>Reference：<a href="https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.MinMaxScaler.html">sklearn.preprocessing.MinMaxScaler</a></li>
  </ul>
</div>

## 介紹模式 & 建立模式

### 門控循環單元-全連接層 (GRU-FC, Gated Recurrent Unit and Fully Connected layer)

#### GRU
<div>
  GRU依序由四條公式組成，各公式都有其名稱及意義
  <ol>
    <li>Update gate：決定有多少長期記憶被使用，用以生成新的長期記憶(final hidden state)</li>
    <li>Reset gate：決定有多少長期記憶被使用，用以生成短期記憶(current hidden state)</li>
    <li>Current hidden state：基於當前的輸入和長期記憶得到的短期記憶</li>
    <li>Final hidden state：結合短期記憶和長期記憶，得到新的長期記憶</li>
    <li>Reference_1：<a href="https://towardsdatascience.com/understanding-gru-networks-2ef37df6c9be">Understanding GRU</a></li>
    <li>Reference_2：<a href="https://clay-atlas.com/blog/2020/06/02/machine-learning-cn-gru-note/">GRU note</a></li>
  </ol>
  理論上，GRU能透過長期記憶(hidden state)的使用，一定程度上記住變數時間序列的週期特徵，但能記住多長時間的週期變化則和實務上GRU的參數選擇相關。
</div>

#### FC
<div>
  FC則是相當簡單的線性變換公式
  <ol>
    <li>y = Ax + b</li>
    <li>Reference：<a href="https://pytorch.org/docs/stable/generated/torch.nn.Linear.html">Linear - Pytorch 2.0 documentations</a></li>
  </ol>
  理論上，FC能透過類似係數矩陣(coefficient matrix)的使用，一定程度上學會變數間的相關性。
</div>

#### GRU-FC
<div>
  
</div>

### 向量自回歸模型 (VAR, Vector AutoRegression)
<div>
  給定一組資料，VAR會透過演算法尋找最佳的係數矩陣，使得誤差向量盡可能的小且能滿足特定條件(<a href="https://en.wikipedia.org/wiki/Vector_autoregression">VAR Wikipedia</a>)。</br>
  一個VAR(p)的公式可寫作</br>
  <ul>
    <li>y(t) = A_1 * y(t-1) + A_2 * y(t-2)、、、 +  A_p * y(t-p) + eta</li>
    <li>y(t)：自相關向量</li>
    <li>A_1、A_2、、、A_p：係數矩陣</li>
    <li>eta：誤差向量</li>
  </ul>
  VAR可視作線性算子，且物理意義上和一維線性回歸雷同。</br>
  <ul>
    <li>一維線性回歸：在一維空間使用最小平方法(least square)讓誤差最小，並找出回歸曲線(regression line)</li>
    <li>VAR：在高維空間中使用OLS(Ordinary Least Square)讓誤差最小，並找出最佳超平面(best-fit hyperplane)</li>
  </ul>
</div>

### 回聲狀態網路 (ESN, Echo State Network)
<div>
</div>
