# QuickStart

https://docs.starrocks.io/docs/quick_start/shared-nothing/

このチュートリアルでは以下について説明します。

- 単一の Docker コンテナで StarRocks を実行する
- データの基本的な変換を含む 2 つの公開データセットをロードする
- SELECT と JOIN を使用してデータを分析する
- 基本的なデータ変換（ETL の T ）

コンテナ準備

```
 sudo docker run -p 9030:9030 -p 8030:8030 -p 8040:8040 -itd \
                             --name quickstart starrocks/allin1-ubuntu
[sudo] masami のパスワード:
Unable to find image 'starrocks/allin1-ubuntu:latest' locally
latest: Pulling from starrocks/allin1-ubuntu
3713021b0277: Pull complete
a29806147caf: Pull complete
9927520c82de: Pull complete
6b71341f0871: Pull complete
bc9cc14b8255: Pull complete
ed2ed7b328b4: Pull complete
4b34d4e23f37: Pull complete
c270d62faf16: Pull complete
e8953b961e15: Pull complete
45e38219cd50: Pull complete
Digest: sha256:99166d73ca70b349829cee3395ce5283a43bbbaa50694cb9f418b6c69c0e31f5
Status: Downloaded newer image for starrocks/allin1-ubuntu:latest
93c9f434fc7beede5778ef252a27eda40a3022d0c403640b2c089e418893a9f2
```

クライアントにログイン  
mysql CLI を使用して、prompt に StarRocks を指定

```
sudo docker exec -it quickstart \
                       mysql -P 9030 -h 127.0.0.1 -u root --
prompt="StarRocks > "
[sudo] masami のパスワード:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.1.0 3.3.1-2b87854

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

```

### いくつかのテーブル作成

```sql
StarRocks > CREATE DATABASE IF NOT EXISTS quickstart;
ckstart;Query OK, 0 rows affected (0.03 sec)

StarRocks >
StarRocks > USE quickstart;
Database changed
```

衝突データテーブル作成

```sql
StarRocks > CREATE TABLE IF NOT EXISTS crashdata (
    ->     CRASH_DATE DATETIME,
    ->     BOROUGH STRING,
    ->     ZIP_CODE STRING,
    ->     LATITUDE INT,
    ->     LONGITUDE INT,
    ->     LOCATION STRING,
    ->     ON_STREET_NAME STRING,
    ->     CROSS_STREET_NAME STRING,
    ->     OFF_STREET_NAME STRING,
    ->     CONTRIBUTING_FACTOR_VEHICLE_1 STRING,
    ->     CONTRIBUTING_FACTOR_VEHICLE_2 STRING,
    ->     COLLISION_ID INT,
    ->     VEHICLE_TYPE_CODE_1 STRING,
    ->     VEHICLE_TYPE_CODE_2 STRING
    -> );
Query OK, 0 rows affected (0.03 sec)
```

気象データテーブル作成

```Sql
StarRocks > CREATE TABLE IF NOT EXISTS weatherdata (
    ->     DATE DATETIME,
    ->     NAME STRING,
    ->     HourlyDewPointTemperature STRING,
    ->     HourlyDryBulbTemperature STRING,
    ->     HourlyPrecipitation STRING,
    ->     HourlyPresentWeatherType STRING,
    ->     HourlyPressureChange STRING,
    ->     HourlyPressureTendency STRING,
    ->     HourlyRelativeHumidity STRING,
    ->     HourlySkyConditions STRING,
    ->     HourlyVisibility STRING,
    ->     HourlyWetBulbTemperature STRING,
    ->     HourlyWindDirection STRING,
    ->     HourlyWindGustSpeed STRING,
    ->     HourlyWindSpeed STRING
    -> );
Query OK, 0 rows affected (0.02 sec)
```

データ挿入

```
masami@masami-L /u/src> curl --location-trusted -u root             \
                                -T ./NYPD_Crash_Data.csv                \
                                -H "label:crashdata-0"                  \
                                -H "column_separator:,"                 \
                                -H "skip_header:1"                      \
                                -H "enclose:\""                         \
                                -H "max_filter_ratio:1"                 \
                                -H "columns:tmp_CRASH_DATE, tmp_CRASH_TIME, CRASH_DATE=str_to_date(conc
at_ws(' ', tmp_CRASH_DATE, tmp_CRASH_TIME), '%m/%d/%Y %H:%i'),BOROUGH,ZIP_CODE,LATITUDE,LONGITUDE,LOCAT
ION,ON_STREET_NAME,CROSS_STREET_NAME,OFF_STREET_NAME,NUMBER_OF_PERSONS_INJURED,NUMBER_OF_PERSONS_KILLED
,NUMBER_OF_PEDESTRIANS_INJURED,NUMBER_OF_PEDESTRIANS_KILLED,NUMBER_OF_CYCLIST_INJURED,NUMBER_OF_CYCLIST
_KILLED,NUMBER_OF_MOTORIST_INJURED,NUMBER_OF_MOTORIST_KILLED,CONTRIBUTING_FACTOR_VEHICLE_1,CONTRIBUTING
_FACTOR_VEHICLE_2,CONTRIBUTING_FACTOR_VEHICLE_3,CONTRIBUTING_FACTOR_VEHICLE_4,CONTRIBUTING_FACTOR_VEHIC
LE_5,COLLISION_ID,VEHICLE_TYPE_CODE_1,VEHICLE_TYPE_CODE_2,VEHICLE_TYPE_CODE_3,VEHICLE_TYPE_CODE_4,VEHIC
LE_TYPE_CODE_5" \
                                -XPUT http://localhost:8030/api/quickstart/crashdata/_stream_load
Enter host password for user 'root':
{
    "TxnId": 2,
    "Label": "crashdata-0",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 423726,
    "NumberLoadedRows": 423725,
    "NumberFilteredRows": 1,
    "NumberUnselectedRows": 0,
    "LoadBytes": 96227746,
    "LoadTimeMs": 2500,
    "BeginTxnTimeMs": 60,
    "StreamLoadPlanTimeMs": 190,
    "ReadDataTimeMs": 1397,
    "WriteDataTimeMs": 2149,
    "CommitAndPublishTimeMs": 100,
    "ErrorURL": "http://127.0.0.1:8040/api/_load_error_log?file=error_log_9f4e8923b95cfd52_9e6ac9589d6e1a92"
}⏎
```

```
masami@masami-L /u/src> curl --location-trusted -u root             \
    -T ./72505394728.csv                    \
    -H "label:weather-0"                    \
    -H "column_separator:,"                 \
    -H "skip_header:1"                      \
    -H "enclose:\""                         \
    -H "max_filter_ratio:1"                 \
    -H "columns: STATION, DATE, LATITUDE, LONGITUDE, ELEVATION, NAME, REPORT_TYPE, SOURCE, HourlyAltimeterSetting, HourlyDewPointTemperature, HourlyDryBulbTemperature, HourlyPrecipitation, HourlyPresentWeatherType, HourlyPressureChange, HourlyPressureTendency, HourlyRelativeHumidity, HourlySkyConditions, HourlySeaLevelPressure, HourlyStationPressure, HourlyVisibility, HourlyWetBulbTemperature, HourlyWindDirection, HourlyWindGustSpeed, HourlyWindSpeed, Sunrise, Sunset, DailyAverageDewPointTemperature, DailyAverageDryBulbTemperature, DailyAverageRelativeHumidity, DailyAverageSeaLevelPressure, DailyAverageStationPressure, DailyAverageWetBulbTemperature, DailyAverageWindSpeed, DailyCoolingDegreeDays, DailyDepartureFromNormalAverageTemperature, DailyHeatingDegreeDays, DailyMaximumDryBulbTemperature, DailyMinimumDryBulbTemperature, DailyPeakWindDirection, DailyPeakWindSpeed, DailyPrecipitation, DailySnowDepth, DailySnowfall, DailySustainedWindDirection, DailySustainedWindSpeed, DailyWeather, MonthlyAverageRH, MonthlyDaysWithGT001Precip, MonthlyDaysWithGT010Precip, MonthlyDaysWithGT32Temp, MonthlyDaysWithGT90Temp, MonthlyDaysWithLT0Temp, MonthlyDaysWithLT32Temp, MonthlyDepartureFromNormalAverageTemperature, MonthlyDepartureFromNormalCoolingDegreeDays, MonthlyDepartureFromNormalHeatingDegreeDays, MonthlyDepartureFromNormalMaximumTemperature, MonthlyDepartureFromNormalMinimumTemperature, MonthlyDepartureFromNormalPrecipitation, MonthlyDewpointTemperature, MonthlyGreatestPrecip, MonthlyGreatestPrecipDate, MonthlyGreatestSnowDepth, MonthlyGreatestSnowDepthDate, MonthlyGreatestSnowfall, MonthlyGreatestSnowfallDate, MonthlyMaxSeaLevelPressureValue, MonthlyMaxSeaLevelPressureValueDate, MonthlyMaxSeaLevelPressureValueTime, MonthlyMaximumTemperature, MonthlyMeanTemperature, MonthlyMinSeaLevelPressureValue, MonthlyMinSeaLevelPressureValueDate, MonthlyMinSeaLevelPressureValueTime, MonthlyMinimumTemperature, MonthlySeaLevelPressure, MonthlyStationPressure, MonthlyTotalLiquidPrecipitation, MonthlyTotalSnowfall, MonthlyWetBulb, AWND, CDSD, CLDD, DSNW, HDSD, HTDD, NormalsCoolingDegreeDay, NormalsHeatingDegreeDay, ShortDurationEndDate005, ShortDurationEndDate010, ShortDurationEndDate015, ShortDurationEndDate020, ShortDurationEndDate030, ShortDurationEndDate045, ShortDurationEndDate060, ShortDurationEndDate080, ShortDurationEndDate100, ShortDurationEndDate120, ShortDurationEndDate150, ShortDurationEndDate180, ShortDurationPrecipitationValue005, ShortDurationPrecipitationValue010, ShortDurationPrecipitationValue015, ShortDurationPrecipitationValue020, ShortDurationPrecipitationValue030, ShortDurationPrecipitationValue045, ShortDurationPrecipitationValue060, ShortDurationPrecipitationValue080, ShortDurationPrecipitationValue100, ShortDurationPrecipitationValue120, ShortDurationPrecipitationValue150, ShortDurationPrecipitationValue180, REM, BackupDirection, BackupDistance, BackupDistanceUnit, BackupElements, BackupElevation, BackupEquipment, BackupLatitude, BackupLongitude, BackupName, WindEquipmentChangeDate" \
    -XPUT http://localhost:8030/api/quickstart/weatherdata/_stream_load


Enter host password for user 'root':
{
    "TxnId": 4,
    "Label": "weather-0",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 22931,
    "NumberLoadedRows": 22931,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 15558550,
    "LoadTimeMs": 279,
    "BeginTxnTimeMs": 0,
    "StreamLoadPlanTimeMs": 10,
    "ReadDataTimeMs": 101,
    "WriteDataTimeMs": 238,
    "CommitAndPublishTimeMs": 29
}⏎
```

ニューヨーク市では 1 時間あたり何件の事故が起きていますか

```sql
StarRocks > USE quickstart;
Database changed
StarRocks > SELECT COUNT(*),
    ->        date_trunc("hour", crashdata.CRASH_DATE) AS Time
    -> FROM crashdata
    -> GROUP BY Time
    -> ORDER BY Time ASC
    -> LIMIT 200;
+----------+---------------------+
| count(*) | Time                |
+----------+---------------------+
|       14 | 2014-01-01 00:00:00 |
|       13 | 2014-01-01 01:00:00 |
・・・
|       14 | 2014-01-09 07:00:00 |
|       55 | 2014-01-09 08:00:00 |
+----------+---------------------+
200 rows in set (0.20 sec)
```

ニューヨークの平均気温はどれくらいですか

```sql

StarRocks > SELECT avg(HourlyDryBulbTemperature),
    ->        date_trunc("hour", weatherdata.DATE) AS Time
    -> FROM weatherdata
    -> GROUP BY Time
    -> ORDER BY Time ASC
    -> LIMIT 100;
+-------------------------------+---------------------+
| avg(HourlyDryBulbTemperature) | Time                |
+-------------------------------+---------------------+
|                            25 | 2014-01-01 00:00:00 |
|                            25 | 2014-01-01 01:00:00 |
・・・
|                            29 | 2014-01-05 01:00:00 |
|                            27 | 2014-01-05 02:00:00 |
|                            28 | 2014-01-05 03:00:00 |
+-------------------------------+---------------------+
100 rows in set (0.07 sec)
```

視界が悪いとき、ニューヨークで運転するのは安全ですか
視界が悪い場合 (0 マイルから 1.0 マイルの間) の衝突事故の数を見てみましょう。この質問に答えるには、DATETIME 列の 2 つのテーブル間で JOIN を使用します。

```sql
StarRocks > SELECT COUNT(DISTINCT c.COLLISION_ID) AS Crashes,
    ->        truncate(avg(w.HourlyDryBulbTemperature), 1) AS Temp_F,
    ->        truncate(avg(w.HourlyVisibility), 2) AS Visibility,
    ->        max(w.HourlyPrecipitation) AS Precipitation,
    ->        date_format((date_trunc("hour", c.CRASH_DATE)), '%d %b %Y %H:%i') AS Hour
    -> FROM crashdata c
    -> LEFT JOIN weatherdata w
    -> ON date_trunc("hour", c.CRASH_DATE)=date_trunc("hour", w.DATE)
    -> WHERE w.HourlyVisibility BETWEEN 0.0 AND 1.0
    -> GROUP BY Hour
    -> ORDER BY Crashes DESC
    -> LIMIT 100;
+---------+--------+------------+---------------+-------------------+
| Crashes | Temp_F | Visibility | Precipitation | Hour              |
+---------+--------+------------+---------------+-------------------+
|     129 |     32 |       0.25 | 0.12          | 03 Feb 2014 08:00 |
|     114 |     32 |       0.25 | 0.12          | 03 Feb 2014 09:00 |
・・・
|      37 |     39 |       0.25 | 0.25          | 18 Jan 2015 15:00 |
|      36 |     60 |       0.75 | 0.05          | 24 May 2014 16:00 |
+---------+--------+------------+---------------+-------------------+
100 rows in set (0.13 sec)
```

凍結した路面での運転はどうでしょうか
水蒸気は華氏 40 度で氷に逆昇華します。このクエリは華氏 0 度から 40 度までの温度を調べます。

```sql
StarRocks > SELECT COUNT(DISTINCT c.COLLISION_ID) AS Crashes,
    ->        truncate(avg(w.HourlyDryBulbTemperature), 1) AS Temp_F,
    ->        truncate(avg(w.HourlyVisibility), 2) AS Visibility,
    ->        max(w.HourlyPrecipitation) AS Precipitation,
    ->        date_format((date_trunc("hour", c.CRASH_DATE)), '%d %b %Y %H:%i') AS Hour
    -> FROM crashdata c
    -> LEFT JOIN weatherdata w
    -> ON date_trunc("hour", c.CRASH_DATE)=date_trunc("hour", w.DATE)
    -> WHERE w.HourlyDryBulbTemperature BETWEEN 0.0 AND 40.5
    -> GROUP BY Hour
    -> ORDER BY Crashes DESC
    -> LIMIT 100;
+---------+--------+------------+---------------+-------------------+
| Crashes | Temp_F | Visibility | Precipitation | Hour              |
+---------+--------+------------+---------------+-------------------+
|     192 |     34 |        1.5 | 0.09          | 18 Jan 2015 08:00 |
|     170 |     21 |       NULL |               | 21 Jan 2014 10:00 |
|     145 |     19 |       NULL |               | 21 Jan 2014 11:00 |
・・・
|      55 |     20 |        0.5 | 0.03          | 05 Mar 2015 16:00 |
|      55 |     38 |         10 | 0.00          | 23 Nov 2015 17:00 |
+---------+--------+------------+---------------+-------------------+
100 rows in set (0.15 sec)
```
