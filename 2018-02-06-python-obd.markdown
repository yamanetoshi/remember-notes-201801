# Python-obd.readthedocs.io 翻訳

このドキュメントは以下のドキュメントの翻訳版となります。

- [python-OBD](http://python-obd.readthedocs.io/en/latest/)

## Getting Started

### Welcome

`Python-OBD` は車の[オンボード診断ポート](https://en.wikipedia.org/wiki/On-board_diagnostics)（OBD-II）からデータを処理するためのライブラリです。 リアルタイムのセンサーデータをストリーミングしたり、エンジン診断のコードを読み取るなどの診断を実行したり、あるいはラズベリーパイに適しています。 このライブラリは、標準の[ELM327 OBD-IIアダプタ](http://www.amazon.com/s/ref=nb_sb_noss?field-keywords=elm327)で動作するように設計されています。

注：`Python-OBD` は 1.0.0 より下にあります。つまり、API がマイナーバージョン間で変更される可能性があります。 アップデートする前に [GitHub のリリース・ページ](https://github.com/brendan-w/python-OBD/releases)で Changelog を調べてください。

### Installation

pypiから最新リリースをインストールします。

```
$ pip install obd
```

注意：Linux で Bluetooth アダプタを使用している場合は、Bluetooth スタックをインストールして設定する必要があります。 Debian ベースのシステムでは、これは通常以下のパッケージをインストールすることを意味します：

```
$ sudo apt-get install bluetooth bluez-utils blueman
```

### Basic Usage

```
import obd

connection = obd.OBD() # auto-connects to USB or RF port

cmd = obd.commands.SPEED # select an OBD command (sensor)

response = connection.query(cmd) # send the command, and parse the response

print(response.value) # returns unit-bearing values thanks to Pint
print(response.value.to("mph")) # user-friendly unit conversions
```

OBD 接続は要求応答方式で動作します。 車からデータを取得するには、必要なデータ（RPM、車速など）を問い合わせるコマンドを送信する必要があります。 `Python-OBD` では、これは `query（）` 関数で行われます。 コマンド自体はオブジェクトとして表され、`obd.commands` の名前または値で参照できます。 `query（）` 関数は、`value` プロパティに解析されたデータを含むレスポンスオブジェクトを返します。

### Module Layout

```
import obd

obd.OBD            # main OBD connection class
obd.Async          # asynchronous OBD connection class
obd.commands       # command tables
obd.Unit           # unit tables (a Pint UnitRegistry)
obd.OBDStatus      # enum for connection status
obd.scan_serial    # util function for manually scanning for OBD adapters
obd.OBDCommand     # class for making your own OBD Commands
obd.ECU            # enum for marking which ECU a command should listen to
obd.logger         # the OBD module's root logger (for debug)
```

### License

GNU General Public License V2

## OBD Connections

ライブラリをインストールしたら、単に `import obd` して、新しい OBD 接続オブジェクトを作成します。 デフォルトでは、`python-OBD` は Bluetooth と USB のシリアルポートを（その順番に）スキャンし、見つかった最初の接続を選択します。 ポートは OBD コンストラクタに接続文字列を渡すことで手動で指定することもできます。 また、`scan_serial` ヘルパーを使用して、接続されているポートのリストを取得することもできます。

```
import obd

connection = obd.OBD() # auto connect

# OR

connection = obd.OBD("/dev/ttyUSB0") # create connection with USB 0

# OR

ports = obd.scan_serial()      # return list of valid USB or RF ports
print ports                    # ['/dev/ttyUSB0', '/dev/ttyUSB1']
connection = obd.OBD(ports[0]) # connect to the first port in the list
```

### OBD(portstr=None, baudrate=None,protocol=None,fast=True)

`portstr`: ご使用のアダプターの UNIX デバイス・ファイルまたは Windows COM ポートを文字列で渡します。 デフォルト値（`None`）はポートを自動的に選択します。

`baudrate`: シリアル接続を設定するボーレートを渡します。 この値ははアダプタによって異なります。 典型的な値は 9600、38400、19200、57600、115200 です。デフォルト値（None）は自動的にボーレートを選択します。

`protocol`: アダプタと通信するときに指定されたプロトコルを　`python-OBD` が強制的に使用するようにします。 可能な値については `protocol_id（）` を参照してください。 デフォルト値（`None`）はプロトコルを自動的に選択します。

`fast`: コマンドを最適化してから車に送ることができます。 `Python-OBD` は現在、このような最適化を2つ行っています。

- キャリッジリターンを送信することで、前のコマンドを繰り返します。
- 応答の制限をコマンドの最後に追加し、N 応答（待機するのではなく最終的にタイムアウトする）を受け取った後でアダプタに返すよう指示します。 この機能は、個々のコマンドに対して有効または無効にすることができます。

高速モードを無効にすると、python-OBDがすべての要求に対して変更されていないコマンドを出力することが保証されます。

### query(command, force=True)

`OBDCommand` を車に送り、`OBDResponse` オブジェクトを返します。 この機能は、車からの応答が受信されるまでブロックされます。 この機能は、指定されたコマンドがあなたの車によってサポートされているかどうかもチェックします。 コマンドがサポートされているとマークされていない場合、コマンドは送信されず、空の `OBDResponse` が返されます。 サポートされていないコマンドを強制的に送信するには、便宜のためにオプションの `force` パラメーターがあります。

ノンブロッキングクエリについては、`Async Querying` を参照してください。

```
import obd
connection = obd.OBD()

r = connection.query(obd.commands.RPM) # returns the response from the car
```

### status()

接続の状態を反映する文字列を値として返します。 これらの値は、`OBDStatus` クラスと比較する必要があります。 それらが文字列であるという事実は、人間の可読性のためだけです。 現在、3つの状態があります。

```
from obd import OBDStatus

# no connection is made
OBDStatus.NOT_CONNECTED # "Not Connected"

# successful communication with the ELM327 adapter
OBDStatus.ELM_CONNECTED # "ELM Connected"

# successful communication with the ELM327 and the vehicle
OBDStatus.CAR_CONNECTED # "Car Connected"
```

中間状態の `ELM_CONNECTED` は、主にエラーを診断するためのものです。 適切な接続が確立されると、この値に遭遇することは決してありません。

### is_connected()

車両との接続が確立されたかどうかを示すブール値を返します。 それは以下と同一です：

```
connection.status() == OBDStatus.CAR_CONNECTED
```

### port_name()

現在接続されているポート名（"`/dev/ttyUSB0`"）の文字列を返します。 接続がない場合、この関数は空の文字列を返します。

### supports(command)

コマンドが車両と　`python-OBD` の両方でサポートされているかどうかを示すブール値を返します。

### protocol_id()
### protocol_name()

どちらの関数も、現在アダプタで使用されているプロトコルの名前を文字列で返します。 プロトコルIDはアダプタで使用される短い値ですが、プロトコル名は人間が判読可能なものです。 `protocol_id（）` 関数は、OBDコンストラクタのプロトコルフィールドで渡す値を検索するのに適しています（ただし、これは主に高度な使い方です）。 これらの関数はシリアルリクエストを行いません。 接続されていない場合、これらの関数は空文字列を返します。 可能な値は次のとおりです。

|ID|Name|
|:-|:---|
|1|SAE J1850 PWM|
|2|SAE J1850 VPW|
|3|AUTO, ISO 9141-2|
|4|ISO 14230-4(KWP 5BAUD)|
|5|ISO 14230-4(KWP FAST)|
|6|ISO 15765-4(CAN 11/500)|
|7|ISO 15765-4(CAN 29/500)|
|8|ISO 15765-4(CAN 11/250)|
|9|ISO 15764-4(CAN 29/250)|
|A|SAE J1939 (CAN 29/250)|

### close()

接続を閉じます。

### supported_commands

車両でサポートされている一連のコマンドを含むプロパティ。

コマンドをサポートされているものとして手動でマークしたい場合（`query（force = True）` を使いたくないとき）、コマンドをこのセットに追加します。 これは `python-OBD` の組み込みコマンドを使用する場合は必要ありませんが、カスタムコマンドを作成する場合には便利です。

```
import obd
connection = obd.OBD()

# manually mark the given command as supported
connection.supported_commands.add(<OBDCommand>)
```

## Command Lookup

`OBDCommand` は、車両から情報を照会するために使用されるオブジェクトです。 これらは、クエリを実行するために必要なすべての情報を含み、車の応答をデコードします。 `Python-OBD` には、最も一般的なコマンド用のテーブルが組み込まれています。 それらは、名前またはモードと PID によって検索することができます。

```
import obd

c = obd.commands.RPM

# OR

c = obd.commands['RPM']

# OR

c = obd.commands[1][12] # mode 1, PID 12 (RPM)
```

`commands` テーブルには、特定の名前または PID が存在するかどうかを判断するためのヘルパーメソッドもあります。

### has_command(command)

指定された `OBDCommand` オブジェクトの存在を内部コマンドテーブルでチェックします。 コマンドは、モードと PID 値で比較されます。

```
import obd
obd.commands.has_command(obd.commands.RPM) # True
```

### has_name(name)

指定された名前のコマンドを内部コマンドテーブルでチェックします。 これは　`in` 演算子の関数でもあります。

```
import obd

obd.commands.has_name('RPM') # True

# OR

'RPM' in obd.commands # True
```

### has_pid(mode, pid)

指定されたモードと　PID　を持つコマンドの内部コマンドテーブルをチェックします。

```
import obd
obd.commands.has_pid(1, 12) # True
```

## Command Tables

### OBD-II adapter (ELM327 commands)

|PID|Name|Description|Response Value|
|:--|:---|:----------|:-------------|
|N/A|ELM_VERSION|OBD-II adapter version string|string|
|N/A|ELM_VOLTAGE|Voltage detected by OBD-II adapter|Unit.volt|

### Mode 01

|PID|Name|Description|Response Value|
|:--|:---|:----------|:-------------|
|00|PIDS_A|Supported PIDs [01-20]|bitarray|
|01|STATUS|Status since DTCs cleared|[special](http://python-obd.readthedocs.io/en/latest/Responses/#status)|
|02|FREEZE_DTC|DTC that triggered the freeze frame|[special](http://python-obd.readthedocs.io/en/latest/Responses/#diagnostic-trouble-codes-dtcs)|
|03|FUEL_STATUS|Fuel System Status|[(string, string)](http://python-obd.readthedocs.io/en/latest/Responses/#fuel-status)|
|04|ENGINE_LOAD|Calculated Engine Load|Unit.percent|
|05|COOLANT_TEMP|Engine Coolant Temperature|Unit.celsius|
|06|SHORT_FUEL_TRIM_1|Short Term Fuel Trim - Bank 1|Unit.percent|
|07|LONG_FUEL_TRIM_1|Long Term Fuel Trim - Bank 1|Unit.percent|
|08|SHORT_FUEL_TRIM_2|Short Term Fuel Trim - Bank 2|Unit.percent|
|0A|FUEL_PRESSURE|Fuel Pressure|Unit.kilopascal|
|0B|INTAKE_PRESSURE|Intake Manifold Pressure|Unit.kilopascal|
|0C|RPM|Engine RPM|Unit.rpm|
|0D|SPEED|Vehicle Speed|Unit.kph|
|0E|TIMING_ADVANCE|Timing Advance|Unit.degree|
|0F|INTAKE_TEMP|Intake Air Temp|Unit.celsius|
|10|MAF|Air Flow Rate (MAF)|Unit.grams_per_second|
|11|THROTTLE_POS|Throttle Position|Unit.percent|
|12|AIR_STATUS|Secondary Air Status|[string](http://python-obd.readthedocs.io/en/latest/Responses/#air-status)|
|13|O2_SENSORS|O2 Sensors Present|[special](http://python-obd.readthedocs.io/en/latest/Responses/#oxygen-sensors-present)|
|14|O2_B1S1|O2: Bank 1 - Sensor 1 Voltage|Unit.volt|
|15|O2_B1S2|O2: Bank 1 - Sensor 2 Voltage|Unit.volt|
|16|O2_B1S3|O2: Bank 1 - Sensor 3 Voltage|Unit.volt|
|17|O2_B1S4|O2: Bank 1 - Sensor 4 Voltage|Unit.volt|
|18|O2_B2S1|O2: Bank 2 - Sensor 1 Voltage|Unit.volt|
|19|O2_B2S2|O2: Bank 2 - Sensor 2 Voltage|Unit.volt|
|1A|O2_B2S3|O2: Bank 2 - Sensor 3 Voltage|Unit.volt|
|1B|O2_B2S4|O2: Bank 2 - Sensor 4 Voltage|Unit.volt|
|1C|OBD_COMPLIANCE|OBD Standards Compliance|string|
|1D|O2_SENSORS_ALT|O2 Sensors Present (alternate)|special|
|1E|AUX_INPUT_STATUS|Auxiliary input status (power take off)|boolean|
|1F|RUN_TIME|Engine Run Time|Unit.second|
|20|PIDS_B|Supported PIDs [21-40]|bitarray|
|21|DISTANCE_W_MIL|Distance Traveled with MIL on|Unit.kilometer|
|22|FUEL_RAIL_PRESSURE_VAC|Fuel Rail Pressure (relative to vacuum)|Unit.kilopascal|
|23|FUEL_RAIL_PRESSURE_DIRECT|Fuel Rail Pressure (direct inject)|Unit.kilopascal|
|24|O2_S1_WR_VOLTAGE|02 Sensor 1 WR Lambda Voltage|Unit.volt|
|25|O2_S2_WR_VOLTAGE|02 Sensor 2 WR Lambda Voltage|Unit.volt|
|26|O2_S3_WR_VOLTAGE|02 Sensor 3 WR Lambda Voltage|Unit.volt|
|27|O2_S4_WR_VOLTAGE|02 Sensor 4 WR Lambda Voltage|Unit.volt|
|28|O2_S5_WR_VOLTAGE|02 Sensor 5 WR Lambda Voltage|Unit.volt|
|29|O2_S6_WR_VOLTAGE|02 Sensor 6 WR Lambda Voltage|Unit.volt|
|2A|O2_S7_WR_VOLTAGE|02 Sensor 7 WR Lambda Voltage|Unit.volt|
|2B|O2_S8_WR_VOLTAGE|02 Sensor 8 WR Lambda Voltage|Unit.volt|
|2C|COMMANDED_EGR|Commanded EGR|Unit.percent|
|2D|EGR_ERROR|EGR Error|Unit.percent|
|2E|EVAPORATIVE_PURGE|Commanded Evaporative Purge|Unit.percent|
|2F|FUEL_LEVEL|Fuel Level Input|Unit.percent|
|30|WARMUPS_SINCE_DTC_CLEAR|Number of warm-ups since codes cleared|Unit.count|
|31|DISTANCE_SINCE_DTC_CLEAR|Distance traveled since codes cleared|Unit.kilometer|
|32|EVAP_VAPOR_PRESSURE|Evaporative system vapor pressure|Unit.pascal|
|33|BAROMETRIC_PRESSURE|Barometric Pressure|Unit.kilopascal|
|34|O2_S1_WR_CURRENT|02 Sensor 1 WR Lambda Current|Unit.milliampere|
|35|O2_S2_WR_CURRENT|02 Sensor 2 WR Lambda Current|Unit.milliampere|
|36|O2_S3_WR_CURRENT|02 Sensor 3 WR Lambda Current|Unit.milliampere|
|37|O2_S4_WR_CURRENT|02 Sensor 4 WR Lambda Current|Unit.milliampere|
|38|O2_S5_WR_CURRENT|02 Sensor 5 WR Lambda Current|Unit.milliampere|
|39|O2_S6_WR_CURRENT|02 Sensor 6 WR Lambda Current|Unit.milliampere|
|3A|O2_S7_WR_CURRENT|02 Sensor 7 WR Lambda Current|Unit.milliampere|
|3B|O2_S8_WR_CURRENT|02 Sensor 8 WR Lambda Current|Unit.milliampere|
|3C|CATALYST_TEMP_B1S1|Catalyst Temperature: Bank 1 - Sensor 1|Unit.celsius|
|3D|CATALYST_TEMP_B2S1|Catalyst Temperature: Bank 2 - Sensor 1|Unit.celsius|
|3E|CATALYST_TEMP_B1S2|Catalyst Temperature: Bank 1 - Sensor 2|Unit.celsius|
|3F|CATALYST_TEMP_B2S2|Catalyst Temperature: Bank 2 - Sensor 2|Unit.celsius|
|40|PIDS_C|Supported PIDs [41-60]|bitarray|
|41|STATUS_DRIVE_CYCLE|Monitor status this drive cycle|[special](http://python-obd.readthedocs.io/en/latest/Responses/#status)|
|42|CONTROL_MODULE_VOLTAGE|Control module voltage|Unit.volt|
|43|ABSOLUTE_LOAD|Absolute load value|Unit.percent|
|44|COMMANDED_EQUIV_RATIO|Commanded equivalence ratio|Unit.ratio|
|45|RELATIVE_THROTTLE_POS|Relative throttle position|Unit.percent|
|46|AMBIANT_AIR_TEMP|Ambient air temperature|Unit.celsius|
|47|THROTTLE_POS_B|Absolute throttle position B|Unit.percent|
|48|THROTTLE_POS_C|Absolute throttle position C|Unit.percent|
|49|ACCELERATOR_POS_D|Accelerator pedal position D|Unit.percent|
|4A|ACCELERATOR_POS_E|Accelerator pedal position E|Unit.percent|
|4B|ACCELERATOR_POS_F|Accelerator pedal position F|Unit.percent|
|4C|THROTTLE_ACTUATOR|Commanded throttle actuator|Unit.percent|
|4D|RUN_TIME_MIL|Time run with MIL on|Unit.minute|
|4E|TIME_SINCE_DTC_CLEARED|Time since trouble codes cleared|Unit.minute|
|4F|unsupported|unsupported	||
|50|MAX_MAF|Maximum value for mass air flow sensor|Unit.grams_per_second|
|51|FUEL_TYPE|Fuel Type|string|
|52|ETHANOL_PERCENT|Ethanol Fuel Percent|Unit.percent|
|53|EVAP_VAPOR_PRESSURE_ABS|Absolute Evap system Vapor Pressure|Unit.kilopascal|
|54|EVAP_VAPOR_PRESSURE_ALT|Evap system vapor pressure|Unit.pascal|
|54|EVAP_VAPOR_PRESSURE_ALT|	Evap system vapor pressure|Unit.pascal|
|55|SHORT_O2_TRIM_B1|Short term secondary O2 trim - Bank 1|Unit.percent|
|56|LONG_O2_TRIM_B1|Long term secondary O2 trim - Bank 1|Unit.percent|
|57|SHORT_O2_TRIM_B2|Short term secondary O2 trim - Bank 2|Unit.percent|
|58|LONG_O2_TRIM_B2|Long term secondary O2 trim - Bank 2|Unit.percent|
|59|FUEL_RAIL_PRESSURE_ABS|Fuel rail pressure (absolute)|Unit.kilopascal|
|5A|RELATIVE_ACCEL_POS|Relative accelerator pedal position|Unit.percent|
|5B|HYBRID_BATTERY_REMAINING|Hybrid battery pack remaining life|Unit.percent|
|5C|OIL_TEMP|Engine oil temperature|Unit.celsius|
|5D|FUEL_INJECT_TIMING|Fuel injection timing|Unit.degree|
|5E|FUEL_RATE|Engine fuel rate|Unit.liters_per_hour|
|5F|unsupported|unsupported||

### Mode 02

Mode 02 コマンドは Mode 01 と同じですが、最後の DTC が発生した時点（フリーズフレーム）からのメトリックです。 名前でアクセスするには、Mode 01 のコマンド名の前にDTC_を付加します。

```
import obd

obd.commands.RPM # the Mode 01 command
# vs.
obd.commands.DTC_RPM # the Mode 02 command
```

### Mode 03

Mode 03 には、車両からのすべての診断トラブルコードを要求する単一のコマンド GET_DTC が含まれています。 レスポンスにはコード自体と記述（`python-OBD` に記述がある場合）が含まれます。 詳細については、[DTC の応答](http://python-obd.readthedocs.io/en/latest/Responses/#diagnostic-trouble-codes-dtcs)セクションを参照してください。

|PID|Name|Description|Response Value|
|:--|:---|:----------|:-------------|
|N/A|GET_DTC|Get Diagnostic Trouble Codes|[special](http://python-obd.readthedocs.io/en/latest/Responses/#diagnostic-trouble-codes-dtcs)|

### Mode 04

|PID|Name|Description|Response Value|
|:--|:---|:----------|:-------------|
|N/A|CLEAR_DTC|Clear DTCs and Freeze data|N/A|

### Mode 06

警告：モード06は実験的なものです。 ソフトウェアテストに合格していますが、実際の車両ではテストされていません。 このモード用のデバッグ出力は非常に高く評価されます。

モード06コマンドは、車両からのさまざまなテスト結果を監視するために使用されます。 このモードのすべてのコマンドは、Monitor Response セクションで説明したのと同じデータ型を返します。 現在、Mode 06 コマンドは CAN プロトコル（ISO 15765-4）に対してのみ実装されています。

表は略

### Mode 07

戻り値は、Mode 03 `GET_DTC`　コマンドと同じ構造体でエンコードされます。

|PID|Name|Description|Response Value|
|:--|:---|:----------|:-------------|
|N/A|GET_CURRENT_DTC|Get DTCs from the current/last driving cycle|[special](http://python-obd.readthedocs.io/en/latest/Responses/#diagnostic-trouble-codes-dtcs)|

## Responses

`query（）` 関数は `OBDResponse` オブジェクトを返します。 これらのオブジェクトには、次のプロパティがあります。

| Property | Description |
|:---------|:------------|
|value|The decoded value from the car|
|command|The `OBDCommand` object that triggered this response|
|message|The internal `Message` object containing the raw response from the car|
|time|Timestamp of response (as given by `time.time()`)|

### is_null()

この機能を使用して、応答が空であるかどうかを確認します。 `Python-OBD` は、車からデータを取得できないときに空の応答を出します。

```
r = connection.query(obd.commands.RPM)

if not r.is_null():
    print(r.value)
```    

### Pint Values

`value` プロパティには通常、Pint `Quantity` オブジェクトが含まれますが、要求に応じて複雑な構造も保持できます。 Pint の量は、値と単位を1つのクラスにまとめ、「4 秒」や「88 mph」などの物理的な値を表すために使用されます。 これにより、数学や単位変換を行う際に一貫性が得られます。 Pint は、`Python-OBD` で `obd.Unit` として公開されている単位のレジストリを管理しています。

以下は、Pint の単位と数量で行える一般的な操作です。 詳細については、Pint Documentation を参照してください。

注：以前のバージョンの `python-OBD` との下位互換性のために、`response.value` の代わりに `response.value.magnitude` を使用してください

```
import obd

>>> response.value
<Quantity(100, 'kph')>

# get the raw python datatype
>>> response.value.magnitude
100

# converts quantities to strings
>>> str(response.value)
'100 kph'

# convert strings to quantities
>>> obd.Unit("100 kph")
<Quantity(100, 'kph')>

# handles conversions nicely
>>> response.value.to('mph')
<Quantity(62.13711922373341, 'mph')>

# scaler math
>>> response.value / 2
<Quantity(50.0, 'kph')>

# non-scaler math requires you to specify units yourself
>>> response.value + (20 * obd.Unit.kph)
<Quantity(120, 'kph')>

# non-scaler math with different units
# handles unit conversions transparently
>>> response.value + (20 * obd.Unit.mph)
<Quantity(132.18688, 'kph')>
```

### Status

status コマンドは、故障インジケータライト（チェックエンジンライト）、スローされるトラブルコードの数、およびエンジンのタイプに関する情報を返します。

```
response.value.MIL              # boolean for whether the check-engine is lit
response.value.DTC_count        # number (int) of DTCs being thrown
response.value.ignition_type    # "spark" or "compression"
```

status　コマンドは、さまざまなシステムテストの可用性とステータスに関する情報も提供します。 これらは　`StatusTest` オブジェクトとして公開され、名前付きプロパティに読み込まれます。 各テストオブジェクトには、可用性と完了のためのブーリアンフラグがあります。

```
response.value.MISFIRE_MONITORING.available    # boolean for test availability
response.value.MISFIRE_MONITORING.complete     # boolean for test completion
```

以下は、python-OBDが報告するすべてのテスト名です：

|Tests|
|:----|
|MISFIRE_MONITORING|
|FUEL_SYSTEM_MONITORING|
|COMPONENT_MONITORING|
|CATALYST_MONITORING|
|HEATED_CATALYST_MONITORING|
|EVAPORATIVE_SYSTEM_MONITORING|
|SECONDARY_AIR_SYSTEM_MONITORING|
|OXYGEN_SENSOR_MONITORING|
|OXYGEN_SENSOR_HEATER_MONITORING|
|EGR_VVT_SYSTEM_MONITORING|
|NMHC_CATALYST_MONITORING|
|NOX_SCR_AFTERTREATMENT_MONITORING|
|BOOST_PRESSURE_MONITORING|
|EXHAUST_GAS_SENSOR_MONITORING|
|PM_FILTER_MONITORING|

### Diagnostic Trouble Codes(DTCs)

各 DTC は、DTC コードと記述（`Python-OBD` に1つがある場合）を含むタプルで表されます。 複数の DTC を返すコマンドの場合は、リストが使用されます。

```
# obd.commands.GET_DTC
response.value = [
    ("P0104", "Mass or Volume Air Flow Circuit Intermittent"),
    ("B0003", ""), # unknown error code, it's probably vehicle-specific
    ("C0123", "")
]

# obd.commands.FREEZE_DTC
response.value = ("P0104", "Mass or Volume Air Flow Circuit Intermittent")
```

### Fuel Status

燃料の状態は、第1および第2の燃料システムの状態を示す2つのストリングのタプルである。 ほとんどの車には1つのシステムしかないので、2番目の要素は空の文字列になる可能性が高いです。 可能な燃料状態は次のとおりです。

|Fuel Status|
|:----------|
|`""`|
|`"Open loop due to insufficient engine temperature"`|
|`"Closed loop, using oxygen sensor feedback to determine fuel mix"`|
|`"Open loop due to engine load OR fuel cut due to deceleration"`|
|`"Open loop due to system failure"`|
|`"Closed loop, using at least one oxygen sensor but there is a fault in the feedback system"`|

### Air Status

空気の状態は、次の文字列のいずれかになります。

|Air Status|
|:---------|
|`"Upstream"`|
|`"Downstream of catalytic converter"`|
|`"From the outside atmosphere or off"`|
|`"Pump commanded on for diagnostics"`|

### Oxygen Sensors Present

センサーの存在のブール値を保持するタプル（バンクとセンサー番号を表す）の 2 次元構造を返します。

```
# obd.commands.O2_SENSORS
response.value = (
    (),                           # bank 0 is invalid, this is merely for correct indexing
    (True,  True,  True,  False), # bank 1
    (False, False, False, False)  # bank 2
)

# obd.commands.O2_SENSORS_ALT
response.value = (
    (),             # bank 0 is invalid, this is merely for correct indexing
    (True,  True),  # bank 1
    (True,  False), # bank 2
    (False, False), # bank 3
    (False, False)  # bank 4
)

# example usage:
response.value[1][2] == True # Bank 1, Sensor 2 is present
```

### Monitors (Mode 06 Responses)

すべての Mode 06コマンドは、要求されたセンサのさまざまなテスト結果を保持する `Monitor` オブジェクトを返します。 1つのモニター応答は、複数のテストを `MonitorTest` オブジェクトの形式で保持できます。 OBD 規格ではいくつかのテストが定義されていますが、車両は常に標準を超えてカスタムテストを実装できます。 `Python-OBD` が認識する標準のテストID（TID）は次のとおりです。

|TID|Name|Description|
|:--|:---|:----------|
|01|RTL_THRESHOLD_VOLTAGE|Rich to lean sensor threshold voltage|
|02|LTR_THRESHOLD_VOLTAGE|Lean to rich sensor threshold voltage|
|03|LOW_VOLTAGE_SWITCH_TIME|Low sensor voltage for switch time calculation|
|04|HIGH_VOLTAGE_SWITCH_TIME|High sensor voltage for switch time calculation|
|05|RTL_SWITCH_TIME|Rich to lean sensor switch time|
|06|LTR_SWITCH_TIME|Lean to rich sensor switch time|
|07|MIN_VOLTAGE|Minimum sensor voltage for test cycle|
|08|MAX_VOLTAGE|Maximum sensor voltage for test cycle|
|09|TRANSITION_TIME|Time between sensor transitions|
|0A|SENSOR_PERIOD|Sensor period|
|0B|MISFIRE_AVERAGE|Average misfire counts for last ten driving cycles|
|0C|MISFIRE_COUNT|Misfire counts for last/current driving cycles|

テスト結果は、プロパティ名またはTID（`obd.commands` 表と同じ）によってアクセスできます。 上記の標準テストはすべて存在しますが、一部のテストでは null になることがあります。 `MonitorTest.is_null（）` 関数を使用して、テストが null かどうかを判別します。

```
response.value.MISFIRE_COUNT

# OR

response.value["MISFIRE_COUNT"]

# OR

response.value[0x0C] # TID for MISFIRE_COUNT
```

すべての `MonitorTest` オブジェクトには、次のプロパティがあります（null テストの場合、`None` に設定されます）

```
result = response.value.MISFIRE_COUNT

result.tid      # integer Test ID for this test
result.name     # test name
result.desc     # test description
result.value    # value of the test (will be a Pint value, or in rare cases, a boolean)
result.min      # maximum acceptable value
result.max      # minimum acceptable value
result.passed   # boolean marking the test as passing
```

エンジンの第2気筒の失火カウントを調べる例を以下に示します。

```
import obd

connection = obd.OBD()

response = connection.query(obd.commands.MONITOR_MISFIRE_CYLINDER_2)

# in the test results, lookup the result for MISFIRE_COUNT
result = response.value.MISFIRE_COUNT

# check that we got data for this test
if not result.is_null():
    print(result.value) # will be a Pint value
else:
    print("Misfire count wasn't reported")
```

## Async Connections

標準の `query（）` 関数はブロックされているため、UIイベントループの危険があります。 これを処理するために、`python-OBD` には、標準の `OBD` オブジェクトの代わりに使用できる `Async` 接続オブジェクトがあります。 `Async` は `OBD` のサブクラスであるため、すべての標準メソッドを継承します。 しかし、`Async` はスレッド更新ループを制御するためにいくつかを追加します。 このループは、あなたのコマンドの値を車両と最新の状態に保ちます。 このようにして、ユーザーが自動車を `query` すると、直ちに最新の応答が返されます。

更新ループは、`start（）` および `stop（）` を呼び出すことによって制御されます。 更新用のコマンドを購読するには、要求された `OBDCommand` で `watch（）` を呼び出します。 更新ループはスレッド化されているため、ループが「停止」している間だけコマンドを「監視」することができます。

```
import obd

connection = obd.Async() # same constructor as 'obd.OBD()'

connection.watch(obd.commands.RPM) # keep track of the RPM

connection.start() # start the async update loop

print connection.query(obd.commands.RPM) # non-blocking, returns immediately
```

コールバックは `watch（）` で指定することもでき、利用可能な場合は新しい `Response` を返します。

```
import obd
import time

connection = obd.Async()

# a callback that prints every new value to the console
def new_rpm(r):
    print r.value

connection.watch(obd.commands.RPM, callback=new_rpm)
connection.start()

# the callback will now be fired upon receipt of new values

time.sleep(60)
connection.stop()
```

### start()

更新ループを開始します。

### stop()

更新ループを停止します。

### paused()

更新ループを一時的に停止するためのコンテキストマネージャ（`with` ステートメント）で使用するヘルパ関数。 これにより、`watch（）` と `unwatch（）` 呼び出しを簡単に保護することができます。 一時停止中に更新ループが実行されていた場合は、コンテキストブロックを終了すると再起動されます。 例えば：

```
with connection.paused() as was_running:
    # connection is stopped within this block
    # your code here
```    

上記のコードは次のものと等価です：

```
was_running = connection.running
connection.stop()

# your code here

if was_running:
    connection.start()
```    

### watch(command, callback=None, force=false)

注意：この関数を呼び出すには、非同期ループを停止または一時停止する必要があります

継続的に更新するコマンドを登録します。 `watch（）` を呼び出した後、`query（）` 関数はそのコマンドから最新の `Response` を返します。 オプションのコールバックも設定することができ、新しい値を受け取ると起動されます。 同じコマンドに対する複数のコールバックは大歓迎です。 オプションの `force` パラメーターを指定すると、サポートされていないコマンドが強制的に送信されます。

### unwatch(command, callback=None)

注意：この関数を呼び出すには、非同期ループを停止または一時停止する必要があります

コマンドの更新を取り消します。 コールバックが指定されていない場合、そのコマンドのすべてのコールバックは破棄されます。 コールバックが指定されている場合、そのコールバックのみが登録解除されます（他はすべてライブ状態のままです）。

### unwatch_all()

注意：この関数を呼び出すには、非同期ループを停止または一時停止する必要があります

すべてのコマンドとコールバックの登録を解除します。

## Custom Commands

必要なコマンドがPython-OBDテーブルにない場合は、新しいOBDCommandオブジェクトを作成できます。 コンストラクタは次の引数を受け取ります（それぞれはプロパティになります）。

|Argument|Type|Description|
|:-------|:---|:----------|
|name|string|(human readability only)|
|desc|string|(human readability only)|
|command|bytes|OBD command in hex (typically mode + PID)|
|bytes|int|Number of bytes expected in response (zero means unknown)|
|decoder|callable|Function used for decoding messages from the OBD adapter|
|ecu(optional)|ECU|ID of the ECU this command should listen to (`ECU.ALL` by default)|
|fast(optional)|bool|Allows python-OBD to after this command for effecieny (`False` by default)|

### Example

```
from obd import OBDCommand, Unit
from obd.protocols import ECU
from obd.utils import bytes_to_int

def rpm(messages):
    """ decoder for RPM messages """
    d = messages[0].data
    v = bytes_to_int(d) / 4.0  # helper function for converting byte arrays to ints
    return v * Unit.RPM # construct a Pint Quantity

c = OBDCommand("RPM", \          # name
               "Engine RPM", \   # description
               b"010C", \        # command
               2, \              # number of return bytes to expect
               rpm, \            # decoding function
               ECU.ENGINE, \     # (optional) ECU filter
               True)             # (optional) allow a "01" to be added for speed
```

デフォルトでは、カスタムコマンドは「車両によってサポートされていません」と扱われます。 これを処理する方法は2つあります。

```
o = obd.OBD()

# use the `force` parameter when querying
o.query(c, force=True)

# OR

# add your command to the set of supported commands
o.supported_commands.add(c)
o.query(c)
```

`OBDCommand`　のあまり直感的ではないフィールドの詳細は次のとおりです。

### OBDCommand.decoder

`decoder` 引数は、以下の形式の関数です。

```
def <name>(<list_of_messages>):
    ...
    return <value>
```

デコーダの戻り値は、`OBDResponse.value` フィールドにロードされます。デコーダには引数として `Message` オブジェクトのリストが与えられます。デコーダが呼び出された場合、このリストには少なくとも1つの `Message` オブジェクトがあることが必要です。各 `Message` オブジェクトには、解析された `bytearray` を保持する `data` プロパティがあります。また、コマンドで指定されたバイト数を取得する必要があります。

注：古いバージョンの `Python-OBD` （デコーダが生の16進文字列を引数として渡した）から移行する場合は、`Message.hex（）` 関数をパッチとして使用できます。

```
def <name>(messages):
    _hex = messages[0].hex()
    ...
    return <value>
```

また、`Message.raw（）` 関数を使用してアダプタから送信された元の文字列にアクセスすることもできます。

### OBDCommand.ecu

`ecu` 引数は、受信メッセージをフィルタリングするために使用される定数です。いくつかのコマンドは、複数の ECU（DTCデコーダなど） を聴くことができ、他のものはエンジン （RPMなど） にのみ関係する場合があります。現在、`python-OBD` はエンジンを区別することしかできませんが、このリストは時間の経過と共に拡張される可能性があります：

- `ECU.ALL`
- `ECU.ALL_KNOWN`
- `ECU.UNKNOWN`
- `ECU.ENGINE`

### OBDCommand.fast

`fast` 引数は、コマンドの最後に "01" を追加するのが安全かどうかを `python-OBD` に伝えます。これにより、アダプターは受信を待つのではなく、受信した最初の応答を返すように指示します（最終的にはタイムアウトに達する）。これはリクエストを大幅に高速化し、ほとんどの `python-OBD` の内部コマンドで有効になります。ただし、異常なコマンドの場合は、これを無効にしておくのが最も安全です。

## Debug

`python-OBD` は python の組み込みログシステムを使用します。 デフォルトでは、stderr に出力を WARNING レベルで送信するように設定されています。 モジュールのロガーは、モジュールのルートにあるロガー変数を介してアクセスできます。 たとえば、すべてのデバッグメッセージのコンソール印刷を有効にするには、次のスニペットを使用します。

```
import obd

obd.logger.setLevel(obd.logging.DEBUG) # enables all debug information
```

または、`python-OBD` からすべてのログ出力を無音にする：

```
import obd

obd.logger.removeHandler(obd.console_handler)
```

## Troubleshooting
### Debug Output

`Python-OBD` が正しく動作していない場合、まずデバッグ出力を有効にする必要があります。 すべてのデバッグ情報をコンソールに出力するには、接続コードの前に次の行を追加します。

```
obd.logger.setLevel(obd.logging.DEBUG)
```

`python-OBD` の一般的なログとその意味は次のとおりです。

### Successful Connection

```
[obd] ========================== python-OBD (v0.4.0) ==========================
[obd] Explicit port defined
[obd] Opening serial port '/dev/pts/2'
[obd] Serial port successfully opened on /dev/pts/2
[obd] write: 'ATZ\r\n'
[obd] wait: 1 seconds
[obd] read: 'ATZ\rELM327 v2.1\r'
[obd] write: 'ATE0\r\n'
[obd] read: 'ATE0\rOK\r'
[obd] write: 'ATH1\r\n'
[obd] read: 'OK\r'
[obd] write: 'ATL0\r\n'
[obd] read: 'OK\r'
[obd] write: 'ATSPA8\r\n'
[obd] read: 'OK\r'
[obd] write: '0100\r\n'
[obd] read: '7E8 06 41 00 FF FF FF FF FC\r'
[obd] write: 'ATDPN\r\n'
[obd] read: 'A8\r'
[obd] Connection successful
[obd] querying for supported PIDs (commands)...
[obd] Sending command: 0100: Supported PIDs [01-20]
[obd] write: '0100\r\n'
[obd] read: '7E8 06 41 00 FF FF FF FF FC\r'
[obd] Sending command: 0120: Supported PIDs [21-40]
[obd] write: '0120\r\n'
[obd] read: '7E8 06 41 20 FF FF FF FF FC\r'
[obd] Sending command: 0140: Supported PIDs [41-60]
[obd] write: '0140\r\n'
[obd] read: '7E8 06 41 40 FF FF FF FE FB\r'
[obd] finished querying with 93 commands supported
[obd] =========================================================================
```

### Unresponsive ELM

```
[obd] ========================== python-OBD (v0.4.0) ==========================
[obd] Explicit port defined
[obd] Opening serial port '/dev/pts/2'
[obd] Serial port successfully opened on /dev/pts/2
[obd] write: 'ATZ\r\n'
[obd] wait: 1 seconds
[obd] __read() found nothing
[obd] __read() found nothing
[obd] __read() never recieved prompt character
[obd] read: ''
[obd] write: 'ATE0\r\n'
[obd] __read() found nothing
[obd] __read() found nothing
[obd] __read() never recieved prompt character
[obd] read: ''
[obd] Connection Error:
[obd]     ATE0 did not return 'OK'
[obd] Failed to connect
[obd] =========================================================================
```

これは、OBD-II アダプタとコンピュータのシリアル接続に問題がある可能性があります。 以下を確認してください：

- Bluetooth デバイスが正しくペア設定されている
- /dev の正しいポートに接続しているか（まったくポートがありません）
- ポートに書き込むための正しいアクセス許可を持っている

`scan_serial（）` ヘルパー関数を使用して、書き込み可能なポートを判別できます。

```
import obd

ports = obd.scan_serial()       # return list of valid USB or RF ports
print ports                    # ['/dev/ttyUSB0', '/dev/ttyUSB1']
```

### Unresponsive Vehicle

```
[obd] ========================== python-OBD (v0.4.0) ==========================
[obd] Explicit port defined
[obd] Opening serial port '/dev/pts/2'
[obd] Serial port successfully opened on /dev/pts/2
[obd] write: 'ATZ\r\n'
[obd] wait: 1 seconds
[obd] read: 'ATZ\rELM327 v2.1\r'
[obd] write: 'ATE0\r\n'
[obd] read: 'ATE0\rOK\r'
[obd] write: 'ATH1\r\n'
[obd] read: 'OK\r'
[obd] write: 'ATL0\r\n'
[obd] read: 'OK\r'
[obd] write: 'ATSPA8\r\n'
[obd] read: 'OK\r'
[obd] write: '0100\r\n'
[obd] read: 'SEARCHING...\rUNABLE TO CONNECT\r'
[obd] write: 'ATDPN\r\n'
[obd] read: '0\r'
[obd] Connection Error:
[obd]     ELM responded with unknown protocol
[obd] Failed to connect
[obd] =========================================================================
```

これは、ELM アダプタと車の接続の問題です。 車両に電源が供給されていること、アダプタと車の OBD-II ポート間の電気的接続は健全であることを確認してください。

