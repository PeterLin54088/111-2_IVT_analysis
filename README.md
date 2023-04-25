# 111-2_IVT_analysis

## 資料製作

### 製作原始資料.nc檔
<div>
  資料來源：ERA5 </br>
  首先，資料的空間是全球範圍，時間是1979-2021，變數是U、V、Q。原始的資料是依據變數類型、時間、高度等，分成了數百份.nc檔。</br>
  為了整合資料並選擇出適當的空間、時間和變數，使用了CDO(Climate Data Operators)做了以下幾件事：</br>
  <ol>
    <li>切割 60E ~ 105E || 0N ~ 25N</li>
    <li>增加 氣壓維度</li>
    <li>黏合 高度維度</li>
    <li>黏合 時間維度</li>
    <li>黏合 U、V、Q變數</li>
  </ol>
  至此，我會得到一個.nc檔，其維度有"時間"、"氣壓"、"緯度"、"經度"，變數為"U"、"V"、"Q"</br>
</div>


### 製作低頻循環.nc檔
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
  考慮到assumption，這裡就不處理長周期波動，只打算移除中周期波動，為此，使用了CDO(Climate Data Operators)做了以下幾件事：</br>
  <ol>
    <li>計算氣候日平均</li>
    <li>計算季節循環 (lowpass filter, f = 3)</li>
  </ol>
  至此，我會得到另一個.nc檔，其格式同上述.nc檔</br>
</div>


## 資料前處理

### 計算IVT & 濾除低頻循環
<div>
  真正期望預報的資料類型為IVT(Integrated water Vapor Transport), 所以我必須計算出IVT，我應當得到兩個變數：原始的IVT 和 季節循環的IVT</br>
  然後，將原始的IVT和季節循環的IVT相減，得到我期望的，主要由短周期波動所組成的filtered IVT</br>
</div>


### 降低維度 (SVD & EOF)
即便我得到了IVT，但這個IVT的維度過大，於是這裡使用SVD(Singular Value Decomposition)將IVT分解為
空間結構：EOF (Empirical Orthogonal Functions)
時間結構：Amplitude time series
特徵值：Singular value
Ref：https://depts.washington.edu/ocean423/notes/EOF.notes.pdf
為了真正降低維度，我僅打算使用部分相對重要的EOF，為此，我以特徵值由大至小為排序，計算各EOF對應到的變異數的解釋力(total variance explained)，並僅使用解釋力高的前數個EOF




