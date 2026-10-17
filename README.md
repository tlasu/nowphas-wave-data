# NOWPHAS 波浪観測データ（Parquet 版）

全国港湾海洋波浪情報網（ナウファス, NOWPHAS）の波浪観測データ（確定値, 1970-2024年）を、
解析しやすい Parquet 形式に変換したデータセット。

出典：全国港湾海洋波浪情報網（ナウファス）（https://nowphas.mlit.go.jp/pastdata/）を加工して作成

本データセットは個人が加工・作成したものであり、国土交通省港湾局・港湾空港技術研究所が作成・提供されたものではない。

## ファイル構成

| パス | 内容 | 粒度 |
|---|---|---|
| `wave_20min/{station_code}.parquet` | 20分値（連続観測, 2004-2024年, 80地点, 約3,208万行） | 地点 x 20分 |
| `wave_2h/{station_code}.parquet` | 2時間値（毎偶数時観測, 1970-2024年, 97地点, 約1,155万行） | 地点 x 2時間 |
| `stations.parquet` / `stations.csv` | 地点メタデータ（名称・機種・水深・緯度経度・データ期間） | 地点 |
| `map.html` | 観測地点の位置マップ。ブラウザで開く | 地点 |
| `station_list_by_year.parquet` | 観測地点一覧PDF（2005-2024年）の年別の内容 | 年 x 地点 |
| `coverage.parquet` | 年ごとの行数と有義波高・波向の有効件数 | 地点 x 種別 x 年 |
| `schema.json` | 列定義（機械可読） | |

`wave_20min` と `wave_2h` は同じ列構成。ディレクトリを指定すれば全地点をまとめて読める。

`map.html` をブラウザで開くと、緯度経度のある地点を地図で見られる。機種と地点名で絞り込める。背景地図は地理院タイルで、表示にはネットワーク接続が必要。同じフォルダに `stations.csv` があるときはそちらを読み、ないときは HTML 内の写しを使う。

## ダウンロード

[Releases](https://github.com/tlasu/nowphas-wave-data/releases) から各ファイルを個別に取得できる。
Release のアセット名にはディレクトリを含められないため、波浪データは `wave_20min_{station_code}.parquet` / `wave_2h_{station_code}.parquet` という名前になっている。

リリースは `.github/workflows/release.yml` で作成する（`v*` タグの push、または Actions 画面からの手動実行）。

## 波浪データの列

| 列 | 型 | 説明 |
|---|---|---|
| `station_code` | string | 地点コード（3桁）。`stations.parquet` と結合するキー |
| `datetime_jst` | timestamp (Asia/Tokyo) | 観測時刻（JST, UTC+9）。2時間値は偶数正時の前後10分ずつ計20分間の観測の中心時刻 |
| `quality_flag` | int16 | 品質フラグ（元値のまま、下表） |
| `n_waves` | int16 | 解析した波の数 |
| `mean_wave_height_m` / `mean_wave_period_s` | float64 | 平均波の波高 [m] / 周期 [s] |
| `significant_wave_height_m` / `significant_wave_period_s` | float64 | 有義波（H1/3, T1/3） |
| `highest_tenth_wave_height_m` / `highest_tenth_wave_period_s` | float64 | 1/10最大波（H1/10, T1/10） |
| `max_wave_height_m` / `max_wave_period_s` | float64 | 最大波（Hmax, Tmax） |
| `wave_direction_deg` | int16 | 主波向 [度]（北=0, 時計回り, 波が来る方向） |
| `is_calm` | bool | 波向が「凪」の場合 true（このとき `wave_direction_deg` は null） |

波高・周期はゼロアップクロス法による。

### 欠測・コード値の扱い

元データの次の値は **null** に置き換えてある。それ以外の値は元データのまま。

| 元の値 | 対象 | 意味 |
|---|---|---|
| `9999` | 波数・波向 | 欠測 |
| `99.99` | 波高 | 欠測 |
| `999.9` | 周期 | 欠測 |
| `66.66` / `77.77` | 波高 | フラグ 66 / 77 の行で求められなかった諸元 |
| `666.6` / `777.7` | 周期 | 同上 |
| `6666` | 波向 | 凪（`is_calm = true`） |

欠測の時刻も行としては存在する（時刻は欠けなく並んでいる）。

### 品質フラグ（`quality_flag`）

| 値 | 意味 |
|---|---|
| 0 | 正常 |
| 4 | 水圧計からの推定 |
| 7 | アナログ記録（全体）からの推定 |
| 77 | アナログ記録（最大波）からの推定。主に有義波高と最大波高のみ値がある |
| 9999 | 欠測 |
| 2, 3, 5, 8, 11, 12, 66 | 公式資料に記載なし（5 は最大波の値がない、66 は値がほぼない） |

フラグの定義は JODC の資料による: https://near-goos1.jodc.go.jp/rdmdb/format/PARI_WAVE.html

## 地点メタデータ（`stations.parquet`）

- 位置・水深・機種は、観測地点一覧PDFでその地点が載っている最新年の値（`location_source_year`）。年による変化は `station_list_by_year.parquet` を参照。
- 2005年より前に観測を終えた11地点（107, 108, 109, 110, 211, 216, 302, 303, 401, 410, 605）は位置情報なし。
- 2007年版のPDFは表を抽出できなかったため `station_list_by_year` に含まれない。
- `wave_20min_first_year` などはデータが存在する最初・最後の年。

## 使い方

```python
import pandas as pd

# 1地点
df = pd.read_parquet("wave_20min/101.parquet")

# 全地点（列を絞ると速い）
df = pd.read_parquet(
    "wave_2h",
    columns=["station_code", "datetime_jst", "significant_wave_height_m"],
)

stations = pd.read_parquet("stations.parquet")
df = df.merge(stations[["station_code", "station_name", "latitude_deg", "longitude_deg"]], on="station_code")
```

```sql
-- DuckDB
SELECT station_code, date_trunc('month', datetime_jst) AS month,
       avg(significant_wave_height_m) AS hs_mean
FROM 'wave_20min/*.parquet'
WHERE quality_flag = 0
GROUP BY ALL ORDER BY ALL;
```

## 注意事項

- 2時間値と20分値は別のデータセット。2004年以降の連続観測地点では、2時間値は20分値と同じ時刻の値になっている（例: 101 の 2010-01-01 00:00 は両方 H1/3 = 4.61 m）。
- 明らかな異常値がごく少数残っている（例: `max_wave_height_m` に 999.99, 89.91、`highest_tenth_wave_height_m` に 990.99、周期 30 s 超が数十行）。解析では `quality_flag = 0` で絞り、物理的に妥当な範囲で除外すること。
- 地点 901 の 2009 年の20分値はサイト側にデータがなく欠けている。
- 611 の 2011 年の20分値はファイルはあるが全行欠測。
- 同名で別コードの地点がある（酒田 102/124、青森 220/224、浜田 305/321、潮岬 301/322、福井 109/117）。観測機器の更新などで地点が移設されたものと考えられるが、位置・水深は異なるので、長期解析で接続する場合は `stations.parquet` で確認すること。
- 2025年以降は NOWPHAS 側で速報値のみの提供のため含めていない。

## 利用条件

元データはナウファスの利用ルール（政府標準利用規約 第2.0版準拠, CC BY 4.0 互換）に従う。
本データセットも [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ja) で提供する。利用時は次の両方を記載すること。

- 出典：全国港湾海洋波浪情報網（ナウファス）（https://nowphas.mlit.go.jp/pastdata/）を加工して作成

論文・報告書等に用いる場合は、ナウファスの案内に従い、例えば次のように記載する。

> 本稿で用いた波浪データは，国土交通省港湾局によって観測され，港湾空港技術研究所で処理されたものである．

データの正確性は保証しない。利用によって生じた結果について、一切の責任を負わない。
