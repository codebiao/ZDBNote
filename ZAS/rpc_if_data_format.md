## gRPC Intetrface Json String Definition
### System parameters
* DF Channel System Parameters
```json
{
    "CmdName" : "SysParam",
    "ForChs": [1,2,3],
    "AlgoBufferPlan": 0,
    "DataRecvTimeout": 60,
    "DataReadyTimeout": 3,
    "CalcZoneRadius": 2500.0,
    "LowLevel" : {
        "SignalBias" : 0,
        "ThresholdMode" : 0,
        "DilationParam" : {
            "KernelShape" : 0,
            "KernelSizeWidth" : 3,
            "KernelSizeHeight" : 3,
            "Iteration" : 5
        },
        "EvtOverloadCount": 50000,
        "HazeCalcMedianLength": 600.0,
        "XencoderAlignPixelPos": 0.0,
        "ConvEnable": 1,
        "SaturatedAdcValue": 4090,
        "EeBoxSizeGear": 0,
        "AccParam": {
            "AccFactor" : 0.1,
            "RootNumAR": 1.0,
        },
        "BpcValue": [1.0, 1.0, 1.0, 1.0]
    },
    "MidLevel" : {
        "EventMergeRSize" : 10.0,
        "EventMergeTSize" : 10.0,
        "CosmicRayThresholdTSize" : 10.0,
        "CosmicRayThresholdRSize" : 5.0,
        "ExtendedAlgoType" : 0,
        "ExtendedThresholdBboxArea" : 40.0,
        "ExtendedThresholdTSize" : 100.0,
        "ExtendedThresholdRSize" : 100.0,
        "CalcFinalSizeEnable" : 0,
        "ImgSize": 224,
        "ImgSaveEnable" : 1,
        "ImgSaveMaxCount" : 1000,
        "ImgSaveAsXbit" : 16,
        "ImgSaveCountPerFile" : 1,
        "RadiusFilterEnable" : 0,
        "RadiusFilterThreshold" : 10.0,
        "RadiusFilterClampToEdgeEnable" : 0
    }
}
```

| Item                  | SubItem   | Type  | Other Desc                                      | Remarks                |
| --------------------- | --------- | ----- | ----------------------------------------------- | ---------------------- |
| DataRecvTimeout       | /         | int   | Data Recieve timeout, unit: ms                  | Important for IMC calc |
| EvtOverloadCount      | /         | int   | Event overload count                            | Important for IMC calc |
| HazeCalcMedianLength  | /         | float | Haze calc median from length                    | Important for IMC calc |
| CalcZoneRadius        | /         | int   | Haze Padding Radius                             | Important for IMC calc |
| XencoderAlignPixelPos | /         | float | Xencoder Align pixel Pos                        | Important for IMC calc |
| ConvEnable            | /         | int   | 2x2 Conv enable flag                            | Important for IMC calc |
| SaturatedAdcValue     | /         | int   | Saturated ADC value                             | Important for IMC calc |
| EeBoxSizeGear         | /         | int   | EE box size gear, 0/1/2 => 14x14/28x28/42x42 px | Important for IMC calc |
| BpcValue              | /         | float | Bpc value, for chosing bpc value                | Important for IMC calc |
| SignalBias            | /         | int   | Signal bias, for chosing signal bias            | Important for IMC calc |
| AccParam              | AccFactor | float | Acc factor, for chosing acc factor              | Important for IMC calc |
| AccParam              | RootNumAR | float | Acc AR, for chosing acc AR                      | Important for IMC calc |


* DIC Channel System Parameters
```json
{
    "CmdName" : "SysParam",
    "ForChs": [4]
}
```

* ACC/EDS/AFS System Parameters
```json
{
    "CmdName" : "SysParam",
    "ForChs": [0],
    "DataRecvTimeout": 60,
    "DataReadyTimeout": 3,
    "Acc":{
        "SinglePmtEventDetectionParam" : {
            "EdMinLen" : 4.0,
            "DwRatio" : 0.5,
            "ThresholdRatio" : 0.6065,
            "MinTriggerEventsCount" : 1,
            "SearchWinSize" : 4
        },
        "AccParameter" : {
            "AccFactor" : 1.0,
            "RootNumberAr" : 1.0,
            "PeakDistance" : 150.0,
            "PeakDistanceDelta" : 0.0,
            "ArFlag" : 0,
            "PpNoiseAdc" : 0,
            "MinHaze" : 0
        },
        "AccEventsOverload" : {
            "TriggerEventOverload" : 100000,
            "AccTriggerCountOverlaod" : 2000
        },
        "SinglePmtTmpParam" : {
            "ScanStartXenc" : 0,
            "ScanStartWenc" : 0,
            "AutoZeroValue" : 0,
            "radius_filter" : 0
        }
    },
    "Afs" : {
    },
    "Eds" : {
        "EdsCameraOffsetWenc" : 0
    }
}
```

| Item                  | SubItem | Type  | Other Desc                         | Remarks                |
| --------------------- | ------- | ----- | ---------------------------------- | ---------------------- |
| DataRecvTimeout       | /       | int   | Data Recieve timeout, unit: ms     | Important for IMC calc |
| EvtOverloadCount      | /       | int   | Event overload count               | Important for IMC calc |
| CalcZoneRadius        | /       | int   | Haze Padding Radius                | Important for IMC calc |
| EdMinLen              | /       | float | tg dir events merge min length     | Important for IMC calc |
| DwRatio               | /       | float | tg dir event peak go down ratio    | Important for IMC calc |
| ThresholdRatio        | /       | float | detection threshold ratio          | Important for IMC calc |
| MinTriggerEvtCount    | /       | int   | Minimum trigger count              | Important for IMC calc |
| SearchWinSize         | /       | int   | For grouping, search events        | Important for IMC calc |
| AccFactor             | /       | float | Acc factor for signal adjust       | Important for IMC calc |
| PeakDistance          | /       | float | main/side beam peak distance       | Important for IMC calc |
| PeakDistanceDelta     | /       | float | main/side beam peak distance delta | Important for IMC calc |
| RootNumberAr          | /       | float | Root number ar for signal adjust   | Important for IMC calc |
| ArFlag                | /       | int   | Ar flag                            | Important for IMC calc |
| P2PNoiseAdc           | /       | int   | For acc region threshold calc      | Important for IMC calc |
| EDSCameraOffsetWenco  | /       | int   | EDS camera wencoder offset         | Important for IMC calc |
| EDSOffCenterThreshold | /       | int   | offcenter threshold                | Important for IMC calc |


### BPC table
* BPC LUT

`BpcTable` is an array. Each item uses explicit fields for `Mode` and `IncMode`, and stores the encoder-indexed BPC values in `EncoderTable`.

Runtime side keeps a map object named `bpc_table` with this index order:

1. `Mode`
2. `IncMode`
3. `ForChs` / channel id
4. `encoder`
5. `vector<float>`

Equivalent C++ shape:

```cpp
std::map<int, std::map<int, std::map<int, std::map<float, std::vector<float>>>>> bpc_table;
```

1. `Mode` as the first-level index
2. `IncMode` as the second-level index
3. `encoder` as the index inside `EncoderTable`
4. the indexed value is a float array

When representing this structure in JSON, all map keys must be strings. That means:

* `encoder` is logically a float index, but it must also be serialized as a string key in JSON, such as `"0.0"`, `"12.5"`

When parsing the JSON, the application should convert the `encoder` key back to a float value before using it as the real index.

Parameter dispatch behavior is independent from `ScanRecipe`, but follows the same per-channel delivery style: one `BpcTable` payload is fanned out by `ForChs`, and each channel updates only its own branch in `bpc_table`.

Update and validation rules:

1. Validation uses the runtime `valid_pixels` stored in cycle's `SingleTdiDfParamTable_t` (`tdidf_info`).
2. Every `SingleTdiDfParam_t.valid_pixels` value in the current `tdidf_info` table must be identical; that common pixel count is used to validate every BPC float array in the payload.
3. If `tdidf_info` is empty, its `valid_pixels` values are inconsistent, or any one vector length does not equal that common `valid_pixels`, the whole payload is rejected.
4. Rejected payloads keep the previous `bpc_table` data unchanged.
5. Repeated deliveries overwrite only the exact matching `(Mode, IncMode, ForChs, encoder)` leaf.
6. Unrelated existing data is preserved and is not deleted.

```json
{
    "CmdName" : "BpcTable",
    "ForChs": [1],
    "BpcTable": [
        {
            "Mode": 0,
            "IncMode": 0,
            "EncoderTable": {
                "1": [1.0, 1.0, 1.0......],
                "10": [1.0, 1.0, 1.0......]
            }
        },
        {
            "Mode": 0,
            "IncMode": 1,
            "EncoderTable": {
                "1": [1.0, 1.0, 1.0......]
            }
        },
        {
            "Mode": 1,
            "IncMode": 0,
            "EncoderTable": {
                "1": [0.31, 0.33, 0.35]
            }
        }
    ]
}
```

| Item | Type | Other Desc | Remarks |
| ---- | ---- | ---------- | ------- |
| BpcTable | array | BPC table entry list | Top-level array |
| BpcTable[i].Mode | int | First-level index value | Explicit field |
| BpcTable[i].IncMode | int | Second-level index value | Explicit field |
| BpcTable[i].EncoderTable | object | Encoder lookup table | Key is `encoder` |
| BpcTable[i].EncoderTable[encoder] | float array | BPC float array | `encoder` is serialized as string in JSON and parsed as float in code; array length must equal the common `valid_pixels` in current `tdidf_info` |
| bpc_table[mode][incmode][for_ch][encoder] | map leaf | Runtime cache object | Leaf value is `vector<float>` |

### Scan Info

* DF Channel Scan Info
```json
{
    "CmdName" : "ScanInfo",
    "ForChs": [1,2,3],
    "TargetFilePath" : "xxx/zpx/",
    "TargetFileName": "yyy.prd",
    "TargetFileName2": "yyy_t2.prd",
    "ChannelId": 0,
    "RecipeId": 0,
    "CollectionName": "CollectionName",
    "WaferId": "from mainPC",
    "TotalScanTimes": 1,
    "CurrentScanTime": 1
}
```
| Item           | SubItem | Type | Other Desc      | Remarks                |
| -------------- | ------- | ---- | --------------- | ---------------------- |
| FilePath       | /       | str  | Full File path  | Important for IMC calc |
| FileName       | /       | str  | File name       | Important for IMC calc |
| FileName2      | /       | str  | T2 file name (DT mode) | Optional |
| RecipeId       | /       | int  | Recipe id       | Important for IMC calc |
| WaferId        | /       | str  | Wafer id        | Important for IMC calc |
| CollectionName | /       | str  | Collection name | Important for IMC calc |
| TotalScanTimes  | /       | int  | 总扫描次数 (1 or 2) | Important for IMC calc |
| CurrentScanTime | /       | int  | 当前扫描次数 (1-based) | Important for IMC calc |

* DIC Channel Scan Info
```json
{
    "CmdName" : "ScanInfo",
    "ForChs": [4],
    "TargetFilePath" : "xxx/zpx/",
    "TargetFileName": "yyy.prd"
}
```

* ACC/EDS/AFS/COMPS/ARCAL/BEAMCAL Channel Scan Info
```json
{
    "CmdName" : "ScanInfo",
    "ForChs": [0],
    "RecipeId": 0,
    "CollectionName": "CollectionName",
    "WaferId": "from mainPC",
    "TotalScanTimes": 1,
    "CurrentScanTime": 1,
    "Acc":{
        "TargetFilePath" : "xxx/zpx/",
        "TargetFileName": "yyy.pad"
    },
    "Comps":{
        "TargetFilePath" : "xxx/zpx/",
        "TargetFileName": "yyy.prd",
        "TargetFileName2": "yyy_t2.prd"
    },
	"Afs":{ 
        "QpdUab" : 1.0,
        "QpdUbb" : 1.0,
        "QpdUcb" : 1.0,
        "QpdUdb" : 0.0,
        "SigmaX0" : 0.1,
        "SigmaY0": 0.1,
        "AfsHeightError": 0.1,
        "QpdTempHeight": 0.1,
        "QpdSigmaXVec" : [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0],
        "QpdHeightVec" : [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0] 
        }
}
```

### Scan Recipe
* DF Channel Scan Recipe
```json
{
    "CmdName" : "ScanRecipe",
    "ForChs": [1,2,3],
    "ScanStartRadius" : 148500,
    "ScanStopRadius" : 50,
    "ScanPitch" : 1000,
    "ScanThreshold" : 200,
    "ScanDualThreshold" : 300,
    "SystemGain": 1.0,
    "HazeCellScale" : {
        "XScale" : 600,
        "YScale" : 600
    },
    "XyCalParam" : {
        "Rotation" : 0.0,
        "RScale" : 1.0,
        "XShift" : 0.0,
        "YShift" : 0.0,
        "DeltaXL" : 0.0,
        "DeltaYL" : 0.0,
        "XEncoFactors" : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0],
        "WEncoFactors" : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
    },
    "NormalCalCurv" : {
        "Nppm" : [1.0, 1.0],
        "Rppm" : [1.0, 1.0]
    },
    "HazeNormalCalCurv" : {
        "Nppm" : [1.0, 1.0],
        "Rppm" : [1.0, 1.0]
    },
    "FilmCurv" : {
        "UmSize" : [1.0 , 1.0],
        "Nppm" : [1.0, 1.0]
    },
    "TputParam" :{
        "ValidPixel" : 1500,
        "CameraPixelStart" : 388,
        "ValidLine" : 2788,
        "FrameTgSize" : 68,
        "FrameRadSize" : 100,
        "PixelTgSize" : 0.7,
        "PixelRadSize" : 0.7,
        "StartRpm" : 130,
        "StopRpm" : 600,
        "Rc": 10000.0,
        "Binning" : 1,
        "LineInfoEnable": 0,
        "ConvPixelRows": 2,
        "ConvPixelCols": 2,
        "ConvPixelKernel": [1.0, 1.0, 1.0, 1.0]
    },
    "BpcEnable" : 0,
    "VarPitchTb" : {
        "XencoderList": [2400, 2430, ......],
        "ValidPixelList": [1500, 1500, ......]
    },
    "HighLevel": {
        "CompositeParams" : {
            "DCO": { "ProximityEps": 50.0 },
            "DCN": { "ProximityEps": 50.0 },
            "DC":  { "ProximityEps": 50.0 },
            "GC":  { "ProximityEps": 50.0 }
        },
        "SliplineParams" : {
            "Enable" : 1,
            "DetectionAngles" : [0.0, 90.0],
            "EdgeIncPercent" : 25.0,
            "SectionWidth" : 200.0,
            "MaxIntraDistance" : 2000.0,
            "MinPointsPerLine" : 10,
            "MinLineLength" : 500.0,
            "MaxLineThickness" : -1.0
        },
        "ClusterParams" : {
            "Enable" : 1,
            "ClusterEps" : 500.0,
            "MinPoints" : 5,
            "OverloadThreshold" : 10000,
            "ExtendedProcMode": 0
        },
        "RecognitionParams" : {
            "Enable" : 1,
            "ScratchEnable" : 1,
            "ScratchAspectRatio" : 5.0,
            "ScratchMinLength" : 180.0,
            "LineEnable" : 1,
            "LineAspectRatio" : 20.0,
            "LineMinLength" : 180.0,
            "LineMaxWidth" : -1.0,
            "RingEnable" : 1,
            "RingMethod" : "ransac",
            "RingRansacIterations" : 100,
            "RingRansacInlierThreshold" : 200.0,
            "RingRansacMinInlierRatio" : 0.4,
            "RingMinRadius" : 10000,
            "RingMinCompleteness" : 0.5
        },
        "MergeParams" : {
            "Enable" : 1,
            "LineMergeEnable" : 0,
            "LineMergeGridSize" : 1000.0,
            "ScratchMergeEnable" : 0,
            "ScratchMergeGridSize" : 1000.0,
            "RingMergeEnable" : 0,
            "RingMergeSectorDeg" : 45.0,
            "AreaMergeEnable" : 0
        }
    }
}
```

| Item          | SubItem | Type | Other Desc            | Remarks                |
| ------------- | ------- | ---- | --------------------- | ---------------------- |
| StartRadius   | /       | int  | Scan start Radius     | Important for IMC calc |
| StopRadius    | /       | int  | Scan stop Radius      | Important for IMC calc |
| ScanPitch     | /       | int  | Scan pitch(fix stage) | Important for IMC calc |
| ScanThreshold | /       | int  | Detection threshold   | Important for IMC calc |
| ScanDualThreshold | / | int | Secondary detection threshold used when Scan command `EnableDualThreshold` is 1 | Optional, default: ScanThreshold |
| HazeCellScale | /       | int  | For haze cell scale   | Important for IMC calc |
| XyCalParam    | /       | int  | For XY correction     | Important for IMC calc |
| NormalCal     | /       | int  | For rppm to nppm      | Important for IMC calc |
| HazeNormalCal | /       | int  | For haze rppm to nppm | Important for IMC calc |
| FilmCurv      | /       | int  | For umSize to nppm    | Important for IMC calc |
| CameraPixelStart | /     | int  | For camera pixel start | Important for IMC calc |
| ConvPixelRows    | /     | int  | Convolution window row count | Important for IMC calc |
| ConvPixelCols    | /     | int  | Convolution window col count | Important for IMC calc |
| BpcEnable       | /     | int  | For bpc enable        | Important for IMC calc |
| **CompositeParams** | | | **Composite 合并参数** | |
| CompositeParams | DCO.ProximityEps | float | DW1O+DW2O+DNO → DCO 合并阈值(μm) | Default: 50.0 |
| CompositeParams | DCN.ProximityEps | float | DW1N+DW2N+DNN → DCN 合并阈值(μm) | Default: 50.0 |
| CompositeParams | DC.ProximityEps  | float | DCN+DCO → DC 合并阈值(μm)，仅双扫描有效 | Default: 50.0 |
| CompositeParams | GC.ProximityEps  | float | DC+DIC(双扫描)或 DCO+DIC(单扫描) → GC 合并阈值(μm) | Default: 50.0 |


* DIC Channel Scan Recipe
```json
{
	"CmdName" : "ScanRecipe",
    "ForChs": [4]
}
```

* ACC/EDS/AFS/HighLevel Channel Scan Recipe
```json
{
    "CmdName" : "ScanRecipe",
    "ForChs": [0],
    "Acc":{
        "ScanStartRadius" : 148500,
        "ScanStopRadius" : 50,
        "ScanPitch" : 1000,
        "PmtThreshold" : 200.0,
        "PmtSystemGain" : 1.0,
        "PmtAccThreshold" : 60.0,
        "PmtPowerCompensate" : 0,
        "PmtA" : 0,
        "PmtB" : 0,
        "PmtC" : 0,
        "PmtD" : 0,
        "HazeCellScale" : {
            "XScale" : 600,
            "YScale" : 600
        },
        "XyCalParam" : {
            "Rotation" : 0.0,
            "RScale" : 1.0,
            "XShift" : 0.0,
            "YShift" : 0.0,
            "DeltaXL" : 0.0,
            "DeltaYL" : 0.0,
            "XEncoFactors" : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0],
            "WEncoFactors" : [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
        },
        "NormalCalCurv" : {
            "Nppm" : [1.0, 1.0],
            "Rppm" : [1.0, 1.0]
        },
        "FilmCurv" : {
            "UmSize" : [1.0 , 1.0],
            "Nppm" : [1.0, 1.0]
        },
        "HazeNormalCalCurv" : {
            "Nppm" : [1.0, 1.0],
            "Rppm" : [1.0, 1.0]
        },
        "TputParam":{
            "StartRpm": 13.46,
            "StopRpm": 134.6,
            "PmtAccSamplingRate": 50000000,
            "PmtAccRoiSize": 30.0,
            "PmtAccWindowSize": 480,
            "PmtAccBeamWidth": 30.0,
            "PmtAccBeamLen": 1200.0,
            "Rc": 10000.0
        },
        "AccDownSampleList": [91, 91, ......],
    },
    "HighLevel":{
        "CompositeParams" : {
            "DCO": { "ProximityEps": 50.0 },
            "DCN": { "ProximityEps": 50.0 },
            "DC":  { "ProximityEps": 50.0 },
            "GC":  { "ProximityEps": 50.0 },
            "SatMaxSizeDW1Um" : 1000.0,
            "SatMaxSizeDW2Um" : 1000.0,
            "SatMaxSizeDNUm"  : 1000.0
        },
        "SliplineParams" : {
            "Enable" : 1,
            "DetectionAngles" : [0.0, 90.0],
            "EdgeIncPercent" : 25.0,
            "SectionWidth" : 200.0,
            "MaxIntraDistance" : 2000.0,
            "MinPointsPerLine" : 10,
            "MinLineLength" : 500.0,
            "MaxLineThickness" : -1.0
        },
        "ClusterParams" : {
            "Enable" : 1,
            "ClusterEps" : 500.0,
            "MinPoints" : 5,
            "OverloadThreshold" : 10000,
            "ExtendedProcMode": 0
        },
        "RecognitionParams" : {
            "Enable" : 1,
            "ScratchEnable" : 1,
            "ScratchAspectRatio" : 5.0,
            "ScratchMinLength" : 180.0,
            "LineEnable" : 1,
            "LineAspectRatio" : 20.0,
            "LineMinLength" : 180.0,
            "LineMaxWidth" : -1.0,
            "RingEnable" : 1,
            "RingMethod" : "ransac",
            "RingRansacIterations" : 100,
            "RingRansacInlierThreshold" : 200.0,
            "RingRansacMinInlierRatio" : 0.4,
            "RingMinRadius" : 10000,
            "RingMinCompleteness" : 0.5
        },
        "MergeParams" : {
            "Enable" : 1,
            "LineMergeEnable" : 0,
            "LineMergeGridSize" : 1000.0,
            "ScratchMergeEnable" : 0,
            "ScratchMergeGridSize" : 1000.0,
            "RingMergeEnable" : 0,
            "RingMergeSectorDeg" : 45.0,
            "AreaMergeEnable" : 0
        },
        "LpdnParams" : {
            "Enable" : 1,                    
            "ChannelRatioEnable" : 1,        
            "WideChannelOption" : 1,         
            "RatioUpperThreshold" : 1.2,     
            "RatioLowerThreshold" : 0.0,     
            "SoloNarrowSkip" : 1,           
            "SoloWideSkip" : 1,              
            "SizeRangeOption" : 0,           
            "WideMinSizeUm" : 0.0,           
            "WideMaxSizeUm" : 9999.0,        
            "NarrowMinSizeUm" : 0.0,         
            "NarrowMaxSizeUm" : 9999.0,
            "RadialZoneEnable" : 0,        
            "LpdnRadialZones" : [            
                { "InnerRadiusUm": 0.0, "OuterRadiusUm": 50000.0 }
            ]
        }
    },
    "Eds":{},
    "Afs":{}
}
```

| Item               | SubItem | Type  | Other Desc              | Remarks                |
| ------------------ | ------- | ----- | ----------------------- | ---------------------- |
| StartRadius        | /       | int   | Scan start Radius       | Important for IMC calc |
| StopRadius         | /       | int   | Scan stop Radius        | Important for IMC calc |
| ScanPitch          | /       | int   | Scan pitch(fix stage)   | Important for IMC calc |
| PmtThreshold       | /       | float | Pmt detection threshold | Important for IMC calc |
| PmtSystemGain      | /       | float | Pmt system gain         | Important for IMC calc |
| PmtAccThreshold    | /       | float | Pmt acc threshold       | Important for IMC calc |
| PmtPowerCompensate | /       | int   | Pmt Power Compensate    | Important for IMC calc |
| PmtA               | /       | int   | clpc coefficient        | Important for IMC calc |
| PmtB               | /       | int   | clpc coefficient        | Important for IMC calc |
| PmtC               | /       | int   | clpc coefficient        | Important for IMC calc |
| PmtD               | /       | int   | clpc coefficient        | Important for IMC calc |
| ExtendedProcMode   | /       | int   | 0: 参与聚类，但是保留Extended的RBN<br>1: 参与聚类但是RBN根据聚类结果进行修改<br>2: 不参与聚类，直接作为Extended，不进行额外处理 | Default: 0 |
| **CompositeParams** | | | **Composite 合并参数** | |
| CompositeParams | DCO.ProximityEps | float | DW1O+DW2O+DNO → DCO 合并阈值(μm) | Default: 50.0 |
| CompositeParams | DCN.ProximityEps | float | DW1N+DW2N+DNN → DCN 合并阈值(μm) | Default: 50.0 |
| CompositeParams | DC.ProximityEps  | float | DCN+DCO → DC 合并阈值(μm)，仅双扫描有效 | Default: 50.0 |
| CompositeParams | GC.ProximityEps  | float | DC+DIC(双扫描)或 DCO+DIC(单扫描) → GC 合并阈值(μm) | Default: 50.0 |
| CompositeParams | SatMaxSizeDW1Um | float | DW1通道缺陷饱和(Sat'd)时尺寸上限(μm) | Default: 1000.0  |
| CompositeParams | SatMaxSizeDW2Um | float | DW2通道缺陷饱和(Sat'd)时尺寸上限(μm) | Default: 1000.0  |
| CompositeParams | SatMaxSizeDNUm  | float | DN通道缺陷饱和(Sat'd)时尺寸上限(μm)  | Default: 1000.0  |
| **LpdnParams**     |         |       | **LPDN 缺陷分类参数** | |
| LpdnParams | Enable              | int   | 是否启用 LPDN 分类步骤；1=启用，0=跳过                                                                        | Default: 1      |
| LpdnParams | ChannelRatioEnable  | int   | 是否启用跨通道尺寸比值判定；1=启用，0=不判定                                                                   | Default: 1      |
| LpdnParams | WideChannelOption   | int   | 宽通道选取：1=强取DW1，2=强取DW2，0=自动取DW1/DW2中尺寸较大者                                                  | Default: 1      |
| LpdnParams | RatioUpperThreshold | float | 上阈值：ratio > 此值 → LPDN；典型配置 1.1~1.3                                                                 | Default: 1.2    |
| LpdnParams | RatioLowerThreshold | float | 下阈值：ratio < 此值 → LPDN；0.0=禁用                                                                         | Default: 0.0    |
| LpdnParams | SoloNarrowSkip      | int   | 仅 narrow 检测到（wide=0）时：1=跳过保留LPD，0=用 WideMinSizeUm 兜底计算比值                                   | Default: 1      |
| LpdnParams | SoloWideSkip        | int   | 仅 wide 检测到（narrow=0）时：1=跳过保留LPD，0=用 NarrowMinSizeUm 兜底计算比值                                 | Default: 1      |
| LpdnParams | SizeRangeOption     | int   | 尺寸范围过滤：0=全量缺陷参与比值判定，1=仅尺寸在 [Min, Max] 区间内的缺陷参与判定                                 | Default: 0      |
| LpdnParams | WideMinSizeUm       | float | wide 通道最小可信尺寸(um)；SoloNarrow 兜底时作比值分母；SizeRangeOption=1 时作范围下界                         | Default: 0.0    |
| LpdnParams | WideMaxSizeUm       | float | wide 通道饱和替换值(um)，wide饱和时替代实测尺寸参与比值计算                                                   | Default: 9999.0 |
| LpdnParams | NarrowMinSizeUm     | float | narrow 通道最小可信尺寸(um)；SoloWide 兜底时作比值分子；SizeRangeOption=1 时作范围下界                         | Default: 0.0    |
| LpdnParams | NarrowMaxSizeUm     | float | narrow 通道饱和替换值(um)，narrow 饱和时替代实测尺寸参与比值计算                                              | Default: 9999.0 |
| LpdnParams | RadialZoneEnable    | int   | 是否启用径向区域规则分箱；1=启用，0=不启用                                                                    | Default: 0      |
| LpdnParams | LpdnRadialZones     | array | LPDN 判定的环形区域列表，仅 RadialZoneEnable=1 时生效；每项含 InnerRadiusUm(内径，um)和OuterRadiusUm(外径，um) | Default: []     |

### Register

```json
{
  	"CmdName" : "Register",
	"LinkChannel": "e_ctrl",
    "ServerIp": "172.16.107.10",
    "ServerPort": 9003
}
```
| Item        | Type   | Other Desc                                        | Remarks |
| ----------- | ------ | ------------------------------------------------- | ------- |
| LinkChannel | String | "ACC", "DW1", "DW2", "DN", "DIC", "COMPS", "CTRL" |         |
|             |        |                                                   |         |

### HeartBeat

```json
{
	"CmdName" : "HeartBeat"
}
```

### QueryStatus

```json
{
	"CmdName" : "QueryStatus"
}
```

### DebugZppj

```json
{
	"CmdName" : "DebugZppj",
    "ForChs": [1,2,3],
    "ZppjContentCode": 3
}
```

### DebugZpmon

```json
{
	"CmdName" : "DebugFpga",
    "ForChs": [1,2,3],
    "ZpmonContentCode": 101
}
```

### RebootFpga

```json
{
	"CmdName" : "RebootFpga",
    "ForChs": [1,2,3]
}
```

### RestartCluster
```json
{
    "CmaName" : "Restart",
    "ForceRestart" : 0
}
```

### Query

- Query zppj status

```json
{
    "CmdName" : "Query",
    "ForChs": [1]
}
```

## Commond

### Scan 

* Scan Job command
```json
{
    "CmdName" : "Scan",
    "ForChs": [0],
    "Mode" : 0,
    "IncMode" : 0,
    "ScanId": "scanid",
    "Sim" : 0,
    "DBG" : 0,
    "EnableDualThreshold" : 0,
    "LowlevelDump": 1,
    "MidlevelDump": 1,
    "HighlevelDump": 1,
    "MaxExecutionTimeMs" : 300000,
    "LowlevelPackageLimitStart" : 0,
    "LowlevelPackageLimitEnd" : 1000000,
    "LowlevelDumpMatsStart" : 0,
    "LowlevelDumpMatsEnd" : -1,
    "LowlevelDumpEventsStart" : 0,
    "LowlevelDumpEventsEnd" : -1,
    "LowlevelDumpHDHazeStart" : 0,
    "LowlevelDumpHDHazeEnd" : -1,
    "LowlevelDumpNormalHazeStart" : 0,
    "LowlevelDumpNormalHazeEnd" : -1,
    "LowlevelDumpCalcHazeMedianStart" : 0,
    "LowlevelDumpCalcHazeMedianEnd" : -1,
    "LowlevelDumpAccStart" : 0,
    "LowlevelDumpAccEnd" : -1,
    "LowlevelDumpBpfData" : 0,
    "SimStartEnable" : 0
}
```

| CmdName | Mode    | IncMode               | Sim                         | DBG                        | ForChs                            | Remarks       |
| ------- | ------- | --------------------- | --------------------------- | -------------------------- | --------------------------------- | ------------- |
| Scan    | 0, UHS_Lite  | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | UHS(Lite) Job       |
| Scan    | 1, HS_Lite   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | HS(Lite) Job        |
| Scan    | 2, ST_Lite   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | ST(Lite) Job        |
| Scan    | 3, HT_Lite   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | HT(Lite) Job        |
| Scan    | 4, UHT_Lite  | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | UHT(Lite) Job       |
| Scan    | 5, UHT2_Lite | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | UHT2(Lite) Job      |
| Scan    | 6, UHS_Full  | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | UHS(Full) Job       |
| Scan    | 7, HS_Full   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | HS(Full) Job        |
| Scan    | 8, ST_Full   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | ST(Full) Job        |
| Scan    | 9, HT_Full   | 0, Oblique, 1, Normal | 0, Real Scan, 1, Simulation | DBG: 0 : Normal, 1 : Debug | Which Channel Execute the command | HT(Full) Job        |
| MaxExecutionTimeMs | / | int | Max execution time, unit: ms | Optional, default: 300000 (5 minutes) | Scan | / |
| LowlevelDumpMatsStart | / | int | Lowlevel dump mats start index | Optional, default: 0 | Scan | / |
| LowlevelDumpMatsEnd | / | int | Lowlevel dump mats end index | Optional, default: -1 | Scan | Scan | / |
| LowlevelDumpEventsStart | / | int | Lowlevel dump events start index | Optional, default: 0 | Scan | / |
| LowlevelDumpEventsEnd | / | int | Lowlevel dump events end index | Optional, default: -1 | Scan | Scan | / |
| LowlevelPackageLimitStart | / | int | Lowlevel package limit start index | Optional, default: 0 | Scan | / |
| LowlevelPackageLimitEnd | / | int | Lowlevel package limit end index | Optional, default: -1 | Scan | Scan | / |
| LowlevelDumpHDHazeStart | / | int | Lowlevel dump HD haze start index | Optional, default: 0 | Scan | / |
| LowlevelDumpHDHazeEnd | / | int | Lowlevel dump HD haze end index | Optional, default: -1 | Scan | Scan | / |
| LowlevelDumpNormalHazeStart | / | int | Lowlevel dump normal haze start index | Optional, default: 0 | Scan | / |
| LowlevelDumpNormalHazeEnd | / | int | Lowlevel dump normal haze end index | Optional, default: -1 | Scan | Scan | / |
| LowlevelDumpBpfData | / | int | Lowlevel dump bpf data | Optional, default: 0 | Scan | / |
| EnableDualThreshold | / | int | 0=disabled, 1=dual threshold mode  | Optional, default: 0 | Scan | / |

### Focus

* Focus Job command
```json
{
    "CmdName" : "Focus",
    "ForChs": [0],
    "Mode" : 0,
    "IncMode" : 0,
    "ScanId": "scanid",
    "Sim" : 0,
    "DBG" : 0,
    "MaxExecutionTimeMs" : 300000,
    "FocalCal": {
        "PeakAdcThreshold": 1000,
        "ImageWidth": 1712,
        "ColBinWidth": 400
    }
}
```

| Item | Type | Default | Other Desc | Remarks |
| --- | --- | --- | --- | --- |
| PeakAdcThreshold | int | 1000 | Min PeakADC for valid particles | Used in AlgoFocalCal |
| ImageWidth | int | 0 | Image column count (pixel width) | 0 = disable ColBins statistics |
| ColBinWidth | int | 400 | Column bin width in pixels | Groups peak_col into intervals of this size; last bin takes the remainder up to ImageWidth |

**ColBins behaviour:**
- `ColBins` statistics are only computed when **both** `ImageWidth > 0` and `ColBinWidth > 0`.
- The number of bins is `ceil(ImageWidth / ColBinWidth)`, capped at 50. The result JSON contains a `BinCount` field equal to `ColBins.length()`. All bins are always reported, including empty ones (`Count = 0`).
- The last bin covers `[(n-1)*ColBinWidth, ImageWidth]` (closed interval).
- Particles with `peak_col >= ImageWidth` are ignored in bin statistics (treated as out-of-range).

### EDS

* EDS Job command
```json
{
    "CmdName" : "EDS",
    "ForChs": [0],
    "Mode" : 0,
    "IncMode" : 0,
    "Sim" : 0,
    "DBG" : 0,
    "EdsMode" : 1,
    "RPS": 2,
    "SamplingRate": 160000,
    "TranNotchImg": 0,
    "NotchrSqure": 0.90,
    "JudgeCover": 1,
    "NotchHeight": 150
}
```

| Command Name | Mode            | IncMode  | Sim                      | DBG                        | Remarks           |
| ------------ | --------------- | -------- | ------------------------ | -------------------------- | ----------------- |
| EDS          | 0, Reflection   | Not Used | 0, normal, 1, simulation | DBG: 0 : Normal, 1 : Debug | Reflection Mode   |
| EDS          | 1, transmission | Not Used | 0, normal, 1, simulation | DBG: 0 : Normal, 1 : Debug | transmission Mode |

### DSNU


* DSNU Job command
```json
{
    "CmdName" : "DSNU",
    "ForChs": [1],
    "StartRadius": 148500,
    "FrameLen": 256,
    "ValidPixels": 2048,
    "Mode" : 0,
    "IncMode" : 0,
    "Sim" : 0,
    "DBG" : 0
}
```

### PRNU

* PRNU Job command
```json
{
    "CmdName" : "PRNU",
    "ForChs": [1],
    "StartRadius": 148500,
    "FrameLen": 256,
    "ValidPixels": 2048,
    "Mode" : 0,
    "IncMode" : 0,
    "Sim" : 0,
    "DBG" : 0
}
```

| Command Name | StartRadius | FrameLen | ValidPixels | ForChs |
| ------------ | ----------- | -------- | ----------- | ------ |
| DSNU         | 148500      | 256      | 2048        | 1      |
| PRNU         | 148500      | 256      | 2048        | 1      |

### Frame

* Frame Mode command
```json
{
    "CmdName" : "Frame",
    "ForChs": [1],
    "Width" : 2276,
    "Height" : 264,
    "FPS" : 30,
    "OperationMode" : 0,
    "Metrics": {
        "Enable": true
    }
}
```

| Command Name | Channel | Width | Height | OperationMode | Metrics | Remarks |
| :----------- | :------ | :---- | :----- | :------------ | :------ | :------ |
| Frame        | DF, DIC | int   | int    | 0: TDI, 1: Area | Optional object ||

### ACCPMT AutoZero
* acc channel single pmt auto-zero command
```json
{
    "CmdName" : "AutoZero",
    "ForChs": [0],
    "Mode" : 0,
    "Sim" : 0,
    "DBG" : 0,
    "CalcSamples" : 10000,
    "ValidateSamples" : 10000
}
```
| Command Name | Channel | Mode | CalcSamples | ValidateSamples | Remarks                                                                             |
| :----------- | :------ | :--- | :---------- | :-------------- | :---------------------------------------------------------------------------------- |
| AutoZero     | ACC     | int  | int         | int             | Mode=0, will do az, but ignore `ValidateSamples`, Mode=1, will do az and validation |
###
### ACCPMT ArCal
* acc channel single pmt ar cal command
```json
{
    "CmdName" : "AccArCal",
    "ForChs": [0],
    "Mode" : 0,
    "Sim" : 0,
    "DBG" : 0
}
```

### ACCPMT BeamCal
* acc channel single pmt beam cal command
```json
{
    "CmdName" : "AccBeamCal",
    "ForChs": [0],
    "Mode" : 0,
    "Sim" : 0,
    "DBG" : 0,
    "PeakDistance": 100.0,
    "PeakDistanceDelta": 10.0,
    "PeakRatio": 10.0,
    "PeakRatioDelta": 5.0
}
```
### MicroView

- MicroView Command

```
{
    "CmdName" : "MicroView",
    "ForChs": [1],
    "ScanId": "scanid",
    "ImgId" : 1768
}
```
### MicroView(ACC Channel)
```
{
    "CmdName" : "MicroView",
    "ForChs": [0],
    "LastScanId":"scanid",
    "PeakIndex" : 12512552225,
    "XEnco" : 20000,
    "WEnco" : 150000,
    "TrackCount" : 25,
    "PointCount" : 2000,
    "Distance" : 1000
}
```
### Composite

- Composite Command: Trigger composite processing to merge multiple scan results into final PRD file

```json
{
    "CmdName" : "Composite"
}
```

| Command Name | Description                                                          | Remarks                                      |
| :----------- | :------------------------------------------------------------------- | :------------------------------------------- |
| Composite    | Merge scan results from multiple passes into final composite PRD    | Triggered after all scan passes are complete |


### WencoCal
* acc channel single pmt and tdi channel wenco cal command
```json
{
    "CmdName" : "WencoCal",
    "ForChs": [0,1,2,3],
    "Mode" : 0,
    "Sim" : 0,
    "DBG" : 0,
    "MinPeakHeight": 5000,
    "MaxPeakHeight": 20000,
    "MinSeparation": 500,
    "SearchRadius": 1000,
    "NumPeaksToUse": 5,
    "GenPkProm": 10000,
    "IsFinal": 0,
    "RPM": 500.0,
    "Radius": 100000.0
}
