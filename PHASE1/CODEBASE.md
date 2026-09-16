# Beckhoff Wafer Inspection Codebase — Complete Technical Extraction

**Project:** KitKat Wafer Placement & Quality Inspection  
**Platform:** Beckhoff TwinCAT 3 (TC1200, TF2000 HMI, TF7100 Vision, TF6280/1 EtherNet/IP)  
**Origin:** Beckhoff Automation UK / York Engineering (Chris Knight)  
**Target Hardware:** Beckhoff C6030-0080 Industrial PC + SICK Ruler-3000 3D Laser Profile Camera  
**Companion Documents:**
- `VISION_CODE_ANALYSIS.md` (Inspection logic flow, review, and manual cross-reference)
- `DELTA_COMPARISON.md` (Ponda URS vs Current Code vs Faclon F.2 Delta Sheet)
- `Wafer_Inspection_Analysis.pdf` (Consolidated report with embedded engineering diagrams)

---

## 1. Codebase Architecture & Two-Tier Split

The Beckhoff TwinCAT project is organized into two distinct sections:

1. **Application Project (`Vanguard_Vision_Inspection`)**:
   - The runnable TwinCAT configuration that deploys to the Beckhoff C6030 IPC runtime.
   - Contains top-level cyclic tasks (`PlcTask` at 10ms cycle, `VisionTask` running on an isolated CPU core).
   - Contains the wiring POUs: `MAIN.TcPOU`, `Vision.TcPOU`, and the global variable list `GVL_Vision.TcGVL`.
   - Maps EtherNet/IP I/O bytes to and from the Rockwell ControlLogix 5580 line PLC.

2. **Compiled Engineering Libraries (`.library`)**:
   - `Nestle_Base.library`: Contains general-purpose Nestlé Vanguard framework components: PackML state models, recipe loaders, and Rockwell EtherNet/IP byte encoders/decoders (`DecodeRockwellData`, `EncodeBeckhoffData`).
   - `Vanguard_Wafer_Inspection.library`: Contains the proprietary vision analysis engine, primarily `ProcessSKU` (with `METHOD Process`) and `Wafer_Inspection_Module`.

### 1.1 TwinCAT editor split: Declaration pane vs Implementation pane

Opening `ProcessSKU.Process` in TwinCAT XAE does **not** show one continuous file. IEC 61131-3 / CODESYS editors always use two independently scrolling panes:

| Pane | Role | `Process` contents |
|---|---|---|
| **Declaration (top)** | Interface and locals only. No executable ST. | `METHOD Process : HRESULT`; `VAR_INPUT Image, hrPrev`; local `VAR i, Pixels, Sum`. |
| **Implementation (bottom)** | Executable Structured Text only. | Calibration, filters, fiducials, `F_VN_*` measurements, pass/fail, cleanup (~670 lines). |

On disk this is the same split inside the `.TcPOU` XML:

```xml
<Method Name="Process">
  <Declaration><![CDATA[
METHOD Process : HRESULT
VAR_INPUT
    Image  : REFERENCE TO CoreVision.ITcVnImage;
    hrPrev : HRESULT;
END_VAR
VAR
    i : UINT;
    Pixels : ULINT;
    Sum : LREAL;
END_VAR
  ]]></Declaration>
  <Implementation>
    <ST><![CDATA[
(* all F_VN_* logic lives here *)
    ]]></ST>
  </Implementation>
</Method>
```

You cannot put `VAR` blocks in the bottom pane, and you cannot put `F_VN_MedianFilter(...)` in the top pane. Instance data (`VisionRecipe`, image buffers) is declared on the function block, not on the method.

### 1.2 Library Pyramid & Design Rationale (from Beckhoff Handover, Aug 13)

Chris Knight (Beckhoff UK) described the architecture as a **pyramid** built for Nestlé reusability:

```
        ┌─────────────────────────┐
        │   Deployment Project    │  ← Hardware-specific (C6030/C6650, camera, tasks)
        │  Vanguard_Vision_Inspection │
        ├─────────────────────────┤
        │  Operational Library    │  ← Wafer inspection logic, state machine, vision
        │ Nestle.Vanguard.WaferInspection │
        ├─────────────────────────┤
        │      Base Library       │  ← Core types, recipes, PackML, EIP encode/decode
        │    Nestle.Vanguard.Base │
        └─────────────────────────┘
```

**Design goals stated by Beckhoff:**
- **Future-proof:** Nestlé can pull `Nestle.Vanguard.Base` into future machines without copying code.
- **Extensible:** New inspection types (e.g., surface detection) can be added as new operational libraries.
- **Supportable:** Beckhoff maintains the libraries; Nestlé/Neebal only modify the deployment layer.
- **Standardized:** PackML state machine ensures consistent behaviour across all Nestlé Vanguard machines.

### 1.3 Recipe Philosophy: Millimetres, Not Pixels

From Giles Roper (Beckhoff UK):

- All recipe dimensions (bar start points, cavity offsets, wafer positions) are stored in **millimetres**.
- A **calibration file** (pixels per mm in X, Y, Z) converts recipe values to machine-specific pixel coordinates at runtime.
- **Why:** The same mould recipe can run on any machine with any camera/resolution. Only the calibration file changes.
- **Capacity:** Up to **100 bars** per mould, **10 wafers** per bar = **1000 possible inspection locations**.

### 1.4 Fiducial Point & Local Reference Levels

- **Fiducial:** Top-left corner of the mould, found by intersecting the top edge (blue line) and left edge (green line).
- **Why:** Compensates for mechanical variation in mould position on the chain.
- **Local reference levels:** For each bar, the code samples height above and below the cavity to establish a local baseline. This compensates for **mould sag** (the mould bows down in the middle under chocolate load).

### 1.5 Vibration Detection (New Feature from York Trials)

- **Observed:** Mould vibration up to **1.7 mm amplitude** causes horizontal lines in the 3D height map.
- **Impact:** Corrupts **delamination analysis** (step-change detection for layer removal).
- **Planned:** Vibration amplitude monitoring with HMI/MES alert when threshold exceeded (~1/3 of layer height, e.g., ~300 µm for Chunky at 2.5 mm/layer).
- **Stretch goal:** Fourier-space filtering to remove vibration without inducing artefacts.

### 1.6 GPU & IPC Context (Phase 2)

- **Phase 2 IPC:** Beckhoff C6650 with **180 W NVIDIA GPU** (procured by Neebal, not Beckhoff).
- **Why GPU:** Low-latency analysis and rejection at Ponda require GPU acceleration for Phase 2 surface inspection.
- **Beckhoff policy:** No third-party GPU bundling due to liability and 5–10 year support constraints. Warranty is not voided by compliant GPU installation.
- **Development acceleration:** Spare C6043 IPC being shipped to Neebal by Caio (Beckhoff).

---

## 2. Hardware Mapping & Global Variables (`GVL_Vision.TcGVL`)

Location: `Vanguard_Vision_Inspection/Inspection_Control/GVLs/GVL_Vision.TcGVL`

```iecst
VAR_GLOBAL
    // Camera connection interface for TwinCAT Vision
    Camera : SimpleCamera;

    // Output results array passed by reference into the vision algorithm
    WaferProcessResults : ARRAY[0..199] OF WaferResultStruct;

    // Core vision processing engine
    WaferDetectionProcess : ProcessSKU(WaferResultArray := WaferProcessResults);

    // ADS Image receiver option for receiving images via ADS
    ADSImageSource : ImageADSArrayReciever<5242880> := (
        Height := 2912, 
        Width := 500, 
        ChannelNumber := 1, 
        ElementType := ETcVnElementType.TCVN_ET_UINT
    );

    // Vision Channel state machine managing camera acquisition and task execution
    ImageChannel : VisionChannel(
        Name := 'Beckhoff Camera', 
        ImageSource := Camera, 
        ImageProcess := WaferDetectionProcess
    ) := (ADSImageOption := ADSImageSource);
END_VAR
```

---

## 3. Cyclic Entry Points

### 3.1 Main Orchestration Program (`MAIN.TcPOU`)
Runs in `PlcTask` (standard cycle time ~10ms). Orchestrates EtherNet/IP communication and module logic.

```iecst
PROGRAM MAIN
VAR
    // Ethernet/IP Inputs (from Rockwell ControlLogix Line PLC)
    EIP_System_Ctrl : SystemControl;
    EIP_System_Ctrl_Byte AT%I* : BYTE;
    DecodeEthernetIPComms : DecodeRockwellData(
        Name := 'Data Decoder', 
        SysCtrl := EIP_System_Ctrl, 
        SysCtrlByte := EIP_System_Ctrl_Byte
    );
	
    // Ethernet/IP Outputs (to Rockwell ControlLogix Line PLC)
    EIP_System_Status : SystemStatus;
    EIP_System_Status_Bytes AT%Q* : ARRAY[0..29] OF BYTE;
    EIP_Wafer_Results_Bytes AT%Q* : ARRAY[0..719] OF BYTE;
    EncodeEthernetIPComms : EncodeBeckhoffData(
        Name := 'Data Encoder',
        SysStatus := EIP_System_Status,
        SysStatusBytes := EIP_System_Status_Bytes,
        WaferResults := GVL_Vision.WaferProcessResults,
        WaferResultBytes := EIP_Wafer_Results_Bytes
    );
	
    // Wafer Detection PackML Module
    WaferVisionModule : Wafer_Inspection_Module(
        Name := 'Wafer Detection Module',
        VisionChannel := GVL_Vision.ImageChannel,
        Process := GVL_Vision.WaferDetectionProcess,
        IO_Ctrl := EIP_System_Ctrl,
        IO_Status := EIP_System_Status,
        WaferResultArray := GVL_Vision.WaferProcessResults
    );

    // Local HMI Interface
    HMI : Wafer_Module_HMI('Vision Inspection', 0, 0);
    Init : BOOL;
END_VAR

// Cyclic calls to modules
DecodeEthernetIPComms.CyclicLogic();
WaferVisionModule.CyclicLogic();
HMI.CyclicLogic();
EncodeEthernetIPComms.CyclicLogic();
```

### 3.2 Vision Processing Task (`Vision.TcPOU`)
Runs in `VisionTask`, assigned to a dedicated, isolated processor core on the C6030 IPC to avoid cycle-jitter.

```iecst
PROGRAM Vision
VAR
END_VAR

// Cyclic call to vision channel operations
// Runs on independent core of the IPC to guarantee real-time execution
GVL_Vision.ImageChannel.CyclicLogic();
```

---

## 4. Vision Processing Engine (`ProcessSKU.TcPOU`)

This is the central function block that executes the wafer inspection algorithm.

### 4.1 Declaration
```iecst
FUNCTION_BLOCK ProcessSKU EXTENDS ImageProcess
VAR
    workingImage : ITcVnImage;
    display : ITcVnDisplayableImage;
    VisionRecipe : WaferDetectionVisionRecipe;
    Result : CoreVision.VisionResult;
    ImageInfo : TcVnImageInfo;
    ValidRecipe : BOOL;
    
    // Internal image buffers
    OriginalImage : ITcVnDisplayableImage;
    medianFilteredImage : ITcVnImage;
    CorrectedImage : ITcVnImage;
    ThresholdImage : ITcVnImage;
    ThresholdInvImage : ITcVnImage;
    ThresholdHighImage : ITcVnImage;
    ThresholdLowImage : ITcVnImage;
    Lvl1Image : ITcVnImage;
    Lvl2Image : ITcVnImage;
    workingDisplayImage : ITcVnImage;
    FinalImage : ITcVnDisplayableImage;
    
    // Diagnostics & Performance Profiling Timers
    TotalTiming : Tc5_Core.PerformanceTimer;
    ScaleTiming : Tc5_Core.PerformanceTimer;
    FiducialsTiming : Tc5_Core.PerformanceTimer;
    MouldHealthTiming : Tc5_Core.PerformanceTimer;
    DisplayTiming : Tc5_Core.PerformanceTimer;
    
    // Results & Diagnostics
    ProcessSKUDiagResult : SKUInternalDiagnosticStruct;
    ProcessWaferResults : REFERENCE TO ARRAY[0..199] OF WaferResultStruct;
END_VAR
```

### 4.2 Recipe Loader Method (`LoadRecipe`)
Validates and applies the vision configuration recipe.

```iecst
METHOD LoadRecipe : HRESULT
VAR_INPUT
    Recipe : WaferDetectionVisionRecipe;
END_VAR

ValidRecipe := FALSE;

// Check filter kernel validity
IF Recipe.VisionProcess.MedianFilterSize = 0 THEN
    LoadRecipe := -1;
    RETURN;
END_IF

// Median filter kernel must be an odd integer (3, 5, etc.)
IF (Recipe.VisionProcess.MedianFilterSize MOD 2) = 0 THEN
    LoadRecipe := -2;
    RETURN;
END_IF

VisionRecipe := Recipe;
ValidRecipe := TRUE;
LoadRecipe := 0;
```

### 4.3 Complete Inspection Execution Method (`Process`)
Full Structured Text implementation extracted from the inspection library:

```iecst
METHOD Process : HRESULT
VAR_INPUT
	Image	: REFERENCE TO CoreVision.ITcVnImage;
	hrPrev : HRESULT;
END_VAR
VAR
	i : uINT;
	Pixels: ULINT;
	Sum: LREAL;
END_VAR

TotalTiming(START := FALSE, RESET := FALSE);
TotalTiming(START := TRUE,RESET := FALSE);

Image.GetImageInfo(ProcessSKUDiagResult.ImageInfo);
//Generate initial display image for testing
hrPrev := F_VN_CopyIntoDisplayableImage(Image, OriginalImage, hrPrev);

//Takes in the raw 16bit height map data form camera source
//calibrates image using the Ruler Scaling and the Pixels/mm value form the recipe
{region "Conversion of the raw 16bit image to calibrated image for processing"}

ScaleTiming(START := FALSE,RESET := FALSE);
ScaleTiming(START := TRUE, RESET := FALSE);
hrPrev := F_VN_MultiplyImageWithScalar(Visionrecipe.Pixels.SickRulerScale/VisionRecipe.Pixels.Z_mm, Image, Image, hrPrev);
hrPrev := F_VN_AddScalarToImage(VisionRecipe.Pixels.SickRulerOffset/VisionRecipe.Pixels.Z_mm, Image, Image, hrPrev);
ScaleTiming(START := FALSE, RESET := FALSE);
ScaleTiming(START := FALSE, RESET := TRUE);	
{endregion}

//Takes the calibrated image
//applies a median filter to remove noise, filter size max is 5 as not usinged values	
{region "Median Filtering reduces the noise in the image"}

hrPrev := F_VN_MedianFilter(Image, medianFilteredImage, VisionRecipe.VisionProcess.MedianFilterSize, hrPrev);

{endregion}

//Takes the calibrated filtered image
//enacts 3 pyramid downs to reduce the image size to a usable size whil epreserving values
//performs an image median operation for 16 bit images
//checks image median is within known sku height ranges
//creates boundary min max mould values for later use.
{region "Derive the Mould Height and Health"}
medianTiming(START := FALSE,RESET := FALSE);
medianTiming(START := TRUE, RESET := FALSE);
hrPrev := F_VN_CopyImage(medianFilteredImage, workingImage, hrPrev);
hrPrev := F_VN_PyramidDown(workingImage, workingImage, hrPrev);
hrPrev := F_VN_PyramidDown(workingImage, workingImage, hrPrev);
hrPrev := F_VN_PyramidDown(workingImage, workingImage, hrPrev);
ImageMedian_Nestle.Execute(workingImage, hrPrev);
ProcessSKUDiagResult.ImageMedianValue := ImageMedian_Nestle.Median;

ProcessSKUDiagResult.MouldInRange := ImageMedian_Nestle.Median > VisionRecipe.Pixels.MinMouldHeight/ VisionRecipe.Pixels.Z_mm AND 
									ImageMedian_Nestle.Median < VisionRecipe.Pixels.MaxMouldHeight/ VisionRecipe.Pixels.Z_mm;
IF ProcessSKUDiagResult.MouldInRange THEN
	ProcessSKUDiagResult.MouldMaxLevel := (1.2*TO_REAL(VisionRecipe.Cavities.EmptyCavityDepth)/VisionRecipe.Pixels.Z_mm) + ImageMedian_Nestle.Median;
	ProcessSKUDiagResult.MouldMinLevel := ImageMedian_Nestle.Median - (1.2*TO_REAL(VisionRecipe.Cavities.EmptyCavityDepth)/VisionRecipe.Pixels.Z_mm);
ELSE
	ProcessSKUDiagResult.MouldMaxLevel := VisionRecipe.Pixels.MaxMouldHeight/ VisionRecipe.Pixels.Z_mm;
	ProcessSKUDiagResult.MouldMinLevel := VisionRecipe.Pixels.MinMouldHeight/ VisionRecipe.Pixels.Z_mm;
END_IF
medianTiming(START := FALSE,RESET := FALSE);
medianTiming(START := FALSE, RESET := TRUE);
{endregion}

//Takes the calibrated filtered image
//applies locate edge operations ot left side and top of image, topline and leftline
//create a flat line across top of image, imageedgeline
//use topline and leftline intersection point for sku fiducial point
//use leftline and imageedgeline to find rotation angle in sku
{region "Find Fiducial point of mould, wafer locations all referenced from this point"}
fiducialTiming(START := FALSE,RESET := FALSE);
fiducialTiming(START := TRUE, RESET := FALSE);
hrPrev := F_VN_CopyImage(medianFilteredImage, workingImage, hrPrev);
hrPrev := F_VN_LocateEdgeExp(workingImage,
								ProcessSKUDiagResult.TopEdgePoints,
								VisionRecipe.SKU.SkuTopEdgeSettings.aStartPoint,
								VisionRecipe.SKU.SkuTopEdgeSettings.aEndPoint,
								VisionRecipe.SKU.SkuTopEdgeSettings.eEdgeDirection,
								VisionRecipe.SKU.SkuTopEdgeSettings.fMinStrength,
								VisionRecipe.SKU.SkuTopEdgeSettings.nSearchLines,
								VisionRecipe.SKU.SkuTopEdgeSettings.fSearchLineDist,
								VisionRecipe.SKU.SkuTopEdgeSettings.nMaxThickness,
								VisionRecipe.SKU.SkuTopEdgeSettings.nSubpixelsIterations,
								VisionRecipe.SKU.SkuTopEdgeSettings.fApproxPrecision,
								VisionRecipe.SKU.SkuTopEdgeSettings.eAlgorithm,
								hrPrev);
								
hrPrev := F_VN_LocateEdgeExp(workingImage,
								ProcessSKUDiagResult.LeftEdgePoints,
								VisionRecipe.SKU.SkuLeftEdgeSettings.aStartPoint,
								VisionRecipe.SKU.SkuLeftEdgeSettings.aEndPoint,
								VisionRecipe.SKU.SkuLeftEdgeSettings.eEdgeDirection,
								VisionRecipe.SKU.SkuLeftEdgeSettings.fMinStrength,
								VisionRecipe.SKU.SkuLeftEdgeSettings.nSearchLines,
								VisionRecipe.SKU.SkuLeftEdgeSettings.fSearchLineDist,
								VisionRecipe.SKU.SkuLeftEdgeSettings.nMaxThickness,
								VisionRecipe.SKU.SkuLeftEdgeSettings.nSubpixelsIterations,
								VisionRecipe.SKU.SkuLeftEdgeSettings.fApproxPrecision,
								VisionRecipe.SKU.SkuLeftEdgeSettings.eAlgorithm,
								hrPrev);
								
hrPrev := F_VN_FitLine(ProcessSKUDiagResult.TopEdgePoints, ProcessSKUDiagResult.TopLine, hrPrev);
hrPrev := F_VN_FitLine(ProcessSKUDiagResult.LeftEdgePoints, ProcessSKUDiagResult.LeftLine, hrPrev);	

	
hrPrev := F_VN_CreateContainer(MainLineContainer, ContainerType_Vector_TcVnPoint2_REAL, 2, hrPrev);

FiducialSearchPoint[0] := 0;
FiducialSearchPoint[1] := 0;
	
hrPrev := F_VN_AddToContainerElements_TcVnPoint2_REAL(FiducialSearchPoint, MainLineContainer, hrPrev);

FiducialSearchPoint[0] := 100;
FiducialSearchPoint[1] := 0;
	
hrPrev := F_VN_AddToContainerElements_TcVnPoint2_REAL(FiducialSearchPoint, MainLineContainer, hrPrev);
	
hrPrev := F_VN_FitLine(MainLineContainer, ImageEdgeLine, hrPrev); 

hrPrev := F_VN_LineIntersectionPoint(ProcessSKUDiagResult.TopLine,
										ProcessSKUDiagResult.LeftLine, 
										ProcessSKUDiagResult.LineIntersectionPoint,
										hrPrev);

hrPrev := F_VN_LineIntersectionPointAndAngle(ProcessSKUDiagResult.TopLine,
										ImageEdgeLine, 
										UnusedIntersection,
										ProcessSKUDiagResult.IntersectionAngle,
										TRUE,
										hrPrev);												
fiducialTiming(START := FALSE,RESET := FALSE);
fiducialTiming(START := FALSE, RESET := TRUE);	
{endregion}

//Takes the calibrated filtered image
//threshold and inv threshold image to obtain reflection pixels
//combine images and add back to original
//threshold and inv threshold to obtain occlusion pixel
//combine images to produce an image with reflection/occlusion removed - corrected image
//check reflection pixels dont exceed a value
//check occlusion pixels dont exceed a value
{region "Occlusion/Reflection pixels removed, clearing invalid regions"}
occlusionTiming(START := FALSE,RESET := FALSE);
occlusionTiming(START := TRUE, RESET := FALSE);
hrPrev := F_VN_CopyImage(medianFilteredImage,ThresholdImage, hrPrev);
hrPrev := F_VN_CopyImage(medianFilteredImage, ThresholdInvImage, hrPrev);
hrPrev := F_VN_CopyImage(medianFilteredImage, workingImage, hrPrev);
	
hrPrev := F_VN_Threshold(ThresholdImage, 
							ThresholdImage, 
							ProcessSKUDiagResult.MouldMaxLevel, 
							ProcessSKUDiagResult.MouldMaxLevel, 
							etcvnthresholdtype.TCVN_TT_BINARY, 
							hrPrev);

hrPrev := F_VN_Threshold(ThresholdInvImage, 
							ThresholdInvImage, 
							ProcessSKUDiagResult.MouldMaxLevel, 
							1.0, 
							etcvnthresholdtype.TCVN_TT_BINARY_INV, 
							hrPrev);	

hrPrev := F_VN_CountNonZeroPixels(ThresholdImage, ProcessSKUDiagResult.reflectionPixels, hrPrev);

hrPrev := F_VN_MultiplyImages(workingImage, ThresholdInvImage, workingImage, hrPrev);

hrPrev := F_VN_AddImages(ThresholdImage, workingImage, workingImage, hrPrev);
	
hrPrev := F_VN_CopyImage(workingImage,ThresholdImage, hrPrev);
hrPrev := F_VN_CopyImage(workingImage, ThresholdInvImage, hrPrev);

hrPrev := F_VN_Threshold(ThresholdInvImage, 
							ThresholdInvImage, 
							ProcessSKUDiagResult.MouldMinLevel, 
							ProcessSKUDiagResult.MouldMinLevel, 
							etcvnthresholdtype.TCVN_TT_BINARY_INV, 
							hrPrev);

hrPrev := F_VN_Threshold(ThresholdImage, 
							ThresholdImage, 
							ProcessSKUDiagResult.MouldMinLevel, 
							1.0, 
							etcvnthresholdtype.TCVN_TT_BINARY, 
							hrPrev);
							
hrPrev := F_VN_CountNonZeroPixels(ThresholdInvImage, ProcessSKUDiagResult.occlusionPixel, hrPrev);

hrPrev := F_VN_MultiplyImages(workingImage, ThresholdImage, workingImage, hrPrev);

hrPrev := F_VN_AddImages(ThresholdInvImage, workingImage, workingImage, hrPrev);
	
ProcessSKUDiagResult.FullImagePixelCount := ProcessSKUDiagResult.ImageInfo.nHeight * ProcessSKUDiagResult.ImageInfo.nWidth;
	
ProcessSKUDiagResult.PercentageReflection := (TO_LREAL(ProcessSKUDiagResult.reflectionPixels)/TO_LREAL(ProcessSKUDiagResult.FullImagePixelCount))*100.0;
ProcessSKUDiagResult.PercentageOcclusion := (TO_LREAL(ProcessSKUDiagResult.occlusionPixel)/TO_LREAL(ProcessSKUDiagResult.FullImagePixelCount))*100.0;
	
ProcessSKUDiagResult.ReflectionOcclusionInRange := (ProcessSKUDiagResult.PercentageReflection < VisionRecipe.VisionProcess.ReflectionMaxLimit)
												AND (ProcessSKUDiagResult.PercentageOcclusion < VisionRecipe.VisionProcess.OcclusionMaxLimit);
 												
hrPrev := F_VN_CopyImage(workingImage,CorrectedImage, hrPrev);												
occlusionTiming(START := FALSE,RESET := FALSE);
occlusionTiming(START := FALSE, RESET := TRUE);
{endregion}	

{region "Inspection of Wafers"}

WaferTiming(START := FALSE, RESET := FALSE);
WaferTiming(START := TRUE, RESET := FALSE);	
	
CurrentWaferIndex := 0;
CurrentBarIndex := 0;
//3D loops iterate over the wafers in each bar, bar in each row and row in SKU	
//info on location stored for sending to Rockwell, to allow rejection
FOR Row := 0 TO ((VisionRecipe.SKU.NumberOfBars/VisionRecipe.SKU.NumberBarsPerRow) - 1) DO
	FOR Column := 0 TO (VisionRecipe.SKU.NumberBarsPerRow - 1) DO
		//Set up Row and Bar data
		//Create target info for wafer rpocessing for both X and Y
		//foiund intersection point + barlocation scaled in dimension
		BarXPositionLevel := TO_UDINT(ProcessSKUDiagResult.LineIntersectionPoint[0] + (visionRecipe.SKU.BarLocations[CurrentBarIndex].X / VisionRecipe.Pixels.X_mm));
		BarXPosition := TO_UDINT(ProcessSKUDiagResult.LineIntersectionPoint[0] + (visionRecipe.SKU.BarLocations[CurrentBarIndex].X / VisionRecipe.Pixels.X_mm));
		BarYPositionLevel := TO_UDINT(ProcessSKUDiagResult.LineIntersectionPoint[1] + (visionRecipe.SKU.BarLocations[CurrentBarIndex].Y / VisionRecipe.Pixels.Y_mm) - LevelsYPositionOffset);
		BarYPosition := TO_UDINT(ProcessSKUDiagResult.LineIntersectionPoint[1] + (visionRecipe.SKU.BarLocations[CurrentBarIndex].Y / VisionRecipe.Pixels.Y_mm));
		
		FOR Cavity := 0 TO (VisionRecipe.SKU.WafersPerBar - 1) DO
			//Set up wafer data 
			//assign the location data for the wafer to wafer result
			WaferResults[CurrentWaferIndex].WaferNumberInMould := CurrentWaferIndex + 1;
			WaferResults[CurrentWaferIndex].ColumnNumber := TO_USINT(Column) + 1;
			WaferResults[CurrentWaferIndex].RowNumber := TO_USINT(Row) + 1;
			WaferResults[CurrentWaferIndex].WaferNumberInBar := TO_USINT(Cavity) + 1;
			
			WaferResults[CurrentWaferIndex].WaferCavityOffset.X := 0.0;
			WaferResults[CurrentWaferIndex].WaferCavityOffset.Y := 0.0;
			WaferResults[CurrentWaferIndex].WaferWidth := 0.0;					
			WaferResults[CurrentWaferIndex].WaferAngleInDegrees := 0.0;
			WaferResults[CurrentWaferIndex].WaferLength := 0.0;										
			WaferResults[CurrentWaferIndex].SDWaferHeight := 0.0;
			WaferResults[CurrentWaferIndex].MeanWaferHeight := 0.0;
			WaferResults[CurrentWaferIndex].WaferLaminatePercentages[0] := 0.0;
			WaferResults[CurrentWaferIndex].WaferLaminatePercentages[1] := 0.0;
			WaferResults[CurrentWaferIndex].WaferLaminatePercentages[2] := 0.0;
			WaferResults[CurrentWaferIndex].WaferLaminatePercentages[3] := 0.0;
			WaferResults[CurrentWaferIndex].WaferStatus := FALSE;
			WaferResults[CurrentWaferIndex].PercentageWaferInCavity := 0.0;
			WaferResults[CurrentWaferIndex].PluggedCavity := FALSE;

//take the target data above and add the individual wafer offset in barsetup with extended window
//take the corrected image
//copy region around the wafer cavity, taking a top and bottom section of SKU with it
//take the top and bottom region and take the median to get the local level around wafer
//calculate the bottom of mould 		
{region "finds the reference levels of the cavity as a reference for wafer detection"}
	
			ProcessWaferResults[CurrentWaferIndex].WaferXPositionLevel := BarXPositionLevel + TO_UDINT(VisionRecipe.Cavities.CavitesOffsets[Cavity].X/VisionRecipe.Pixels.X_mm);
			ProcessWaferResults[CurrentWaferIndex].WaferYPositionLeveL := BarYPositionLevel + TO_UDINT(VisionRecipe.Cavities.CavitesOffsets[Cavity].Y/VisionRecipe.Pixels.Y_mm);
			hrPrev := F_VN_CopyImageRegion(CorrectedImage,
											ProcessWaferResults[CurrentWaferIndex].WaferXPositionLevel,
											ProcessWaferResults[CurrentWaferIndex].WaferYPositionLeveL,
											TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
											TO_UDINT((VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm) + LevelsWaferLengthExtension), 
											workingImage, 
											hrPrev);

			hrPrev := F_VN_CopyImageRegion(workingImage,
											TO_UDINT(visionrecipe.Cavities.LevelReference1.Offset.X),
											TO_UDINT(visionrecipe.Cavities.LevelReference1.Offset.Y),
											TO_UDINT(visionrecipe.Cavities.LevelReference1.Size.X - visionrecipe.Cavities.LevelReference1.Offset.X),
											TO_UDINT(visionrecipe.Cavities.LevelReference1.Size.Y - visionrecipe.Cavities.LevelReference1.Offset.Y), 
											Lvl1Image, 
											hrPrev);

			hrPrev := F_VN_CopyImageRegion(workingImage,
											TO_UDINT(visionrecipe.Cavities.LevelReference2.Offset.X),
											TO_UDINT(visionrecipe.Cavities.LevelReference2.Offset.Y),
											TO_UDINT(visionrecipe.Cavities.LevelReference2.Size.X - visionrecipe.Cavities.LevelReference2.Offset.X),
											TO_UDINT(visionrecipe.Cavities.LevelReference2.Size.Y - visionrecipe.Cavities.LevelReference2.Offset.Y),  
											Lvl2Image, 
											hrPrev);
					
			ImageMedian_Nestle.Execute(Lvl1Image, hrPrev);
			ProcessWaferResults[CurrentWaferIndex].LevelRef1Median := ImageMedian_Nestle.Median;
			ImageMedian_Nestle.Execute(Lvl2Image, hrPrev);
			ProcessWaferResults[CurrentWaferIndex].LevelRef2Median := ImageMedian_Nestle.Median;
			
			ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel := (ProcessWaferResults[CurrentWaferIndex].LevelRef1Median + ProcessWaferResults[CurrentWaferIndex].LevelRef2Median)/2.0;
			
			ProcessWaferResults[CurrentWaferIndex].DifferenceInRefLevel := (ABS(ProcessWaferResults[CurrentWaferIndex].LevelRef1Median - ProcessWaferResults[CurrentWaferIndex].LevelRef2Median))*VisionRecipe.Pixels.Z_mm;
			ProcessWaferResults[CurrentWaferIndex].RefLevelInRange := 	ProcessWaferResults[CurrentWaferIndex].DifferenceInRefLevel < VisionRecipe.Cavities.MaxDifferenceBetweenLevels;

{endregion}	
		

//take the target data above and add the individual wafer offset in barsetup with extended window
//take the corrected image
//copy region around the wafer cavity
//threshold around the levels from above, taking pixels above min and pixels above max
//uses these pixel counts to check number of pixles above percentage for plugged cavity
//use these pixel count to check number of pixles above percentage for passing wafer
{region "Calculate the percentage of cavity taken by wafer to calculate pass/fail"}
	
			ProcessWaferResults[CurrentWaferIndex].WaferXPosition := BarXPosition + TO_UDINT(VisionRecipe.Cavities.CavitesOffsets[Cavity].X/VisionRecipe.Pixels.X_mm);
			ProcessWaferResults[CurrentWaferIndex].WaferYPosition := BarYPosition + TO_UDINT(VisionRecipe.Cavities.CavitesOffsets[Cavity].Y/VisionRecipe.Pixels.Y_mm);
			hrPrev := F_VN_CopyImageRegion(CorrectedImage,
									ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
									ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
									TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
									TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm), 
									workingImage, 
									hrPrev);
			workingImage.GetImageInfo(ProcessWaferResults[CurrentWaferIndex].ImageInfo);
			ProcessWaferResults[CurrentWaferIndex].PixelCount := ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight * ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth;						
			ProcessWaferResults[CurrentWaferIndex].LowThresholdLevel := ((ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel - visionRecipe.Cavities.EmptyCavityDepth/VisionRecipe.Pixels.Z_mm) + VisionRecipe.Wafer.MinWaferHeightFromBase/VisionRecipe.Pixels.Z_mm);
			ProcessWaferResults[CurrentWaferIndex].HighTresholdLevel := ((ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel - visionRecipe.Cavities.EmptyCavityDepth/VisionRecipe.Pixels.Z_mm) + VisionRecipe.Wafer.MaxWaferHeightFromBase/VisionRecipe.Pixels.Z_mm);
			ProcessWaferResults[CurrentWaferIndex].PluggedThresholdLevel := ((ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel - VisionRecipe.Wafer.PlugDistanceFromMouldToSurface/VisionRecipe.Pixels.Z_mm)); 
			
			hrPrev := F_VN_Threshold(workingImage, 
										ThresholdHighImage,
										ProcessWaferResults[CurrentWaferIndex].PluggedThresholdLevel,
										ThresholdMaxLevel,
										ETcVnThresholdType.TCVN_TT_BINARY,
										hrPrev);

			hrPrev := F_VN_CountNonZeroPixels(ThresholdHighImage, ProcessWaferResults[CurrentWaferIndex].PixelsAbovePlugLevel, hrPrev);
			
			WaferResults[CurrentWaferIndex].PluggedCavity := TO_LREAL(ProcessWaferResults[CurrentWaferIndex].PixelsAbovePlugLevel)/ProcessWaferResults[CurrentWaferIndex].PixelCount*100.0 > TO_LREAL(VisionRecipe.Wafer.PluggedCavityPercentage);
			
			
			hrPrev := F_VN_Threshold(workingImage, 
										ThresholdLowImage,
										ProcessWaferResults[CurrentWaferIndex].LowThresholdLevel, 
										ThresholdMaxLevel,
										ETcVnThresholdType.TCVN_TT_BINARY,
										hrPrev);
			//hrPrev := F_VN_CountNonZeroPixels(ThresholdLowImage, ProcessWaferResults[CurrentWaferIndex].PixelsAboveMinLevel, hrPrev);
			
										
			hrPrev := F_VN_Threshold(workingImage, 
										ThresholdHighImage,
										ProcessWaferResults[CurrentWaferIndex].HighTresholdLevel,
										ThresholdMaxLevel,
										ETcVnThresholdType.TCVN_TT_BINARY,
										hrPrev);
			hrPrev := F_VN_SubtractImages(ThresholdLowImage,ThresholdHighImage, workingImage, hrPrev);

			hrPrev := F_VN_CountNonZeroPixels(workingImage, ProcessWaferResults[CurrentWaferIndex].PixelsAboveMaxLevel, hrPrev);
			
			ProcessSKUDiagResult.IdealWaferArea := (WaferIdealHeight / VisionRecipe.Pixels.Y_mm) * (WaferIdealWidth / VisionRecipe.Pixels.X_mm);
			
			WaferResults[CurrentWaferIndex].PercentageWaferInCavity := TO_LREAL(ProcessWaferResults[CurrentWaferIndex].PixelsAboveMaxLevel)/ProcessSKUDiagResult.IdealWaferArea * 100.0;
			IF WaferResults[CurrentWaferIndex].PercentageWaferInCavity > VisionRecipe.Wafer.PassingWaferCavityPercentage THEN
				ProcessWaferResults[CurrentWaferIndex].WaferPercentageOK := TRUE;
			ELSE
				ProcessWaferResults[CurrentWaferIndex].WaferPercentageOK := FALSE;
			END_IF
{endregion}
		

//take the corrected image
//copy region around the wafer cavity
//perform a blob detection to find wafer sensor
//use edge detection of left and top edge of wafer
//extract the length and width and scale to real mm values
{region "check Wafer Dimensions and Angle"}
			IF NOT WaferResults[CurrentWaferIndex].PluggedCavity THEN
				hrPrev := F_VN_CopyImageRegion(CorrectedImage,
												ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
												ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
												TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
												TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm), 
												workingImage, 
												hrPrev);
	
				ProcessWaferResults[CurrentWaferIndex].WaferIdealLevel := ((ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel - visionRecipe.Cavities.EmptyCavityDepth/VisionRecipe.Pixels.Z_mm) + VisionRecipe.Wafer.HeightFromMouldBase/VisionRecipe.Pixels.Z_mm);
				hrPrev := F_VN_Threshold(workingImage, 
										ThresholdImage, 
										ProcessWaferResults[CurrentWaferIndex].WaferIdealLevel, 
										ThresholdMaxLevel, 
										etcvnthresholdtype.TCVN_TT_BINARY, 
										hrPrev);
				hrPrev := F_VN_ConvertElementType(ThresholdImage, workingimage, etcvnelementtype.TCVN_ET_USINT, hrPrev);
			
				BlobDetectParams.fMinArea := (TO_REAL(VisionRecipe.Wafer.PassingWaferCavityPercentage) / RequiredPassingPixelFactor) * TO_REAL(ProcessWaferResults[CurrentWaferIndex].PixelCount);
				
				hrPrev := F_VN_DetectBlobs(workingImage, WaferBlobsDetectedInCavity, blobDetectParams, hrPrev);
				
				hrPrev := F_VN_GetNumberOfElements(WaferBlobsDetectedInCavity, NumberOfWafersDetectedInCavity, hrPrev);
				
				IF NumberOfWafersDetectedInCavity >= 1 THEN
					hrPrev := F_VN_GetAt_ITcVnContainer(WaferBlobsDetectedInCavity, DetectedWaferOutline, 0,hrPrev);
					
					hrPrev := F_VN_ContourCenterOfMass(DetectedWaferOutline, ProcessWaferResults[CurrentWaferIndex].WaferCoM, hrPrev);
					
					WaferResults[CurrentWaferIndex].WaferCavityOffset.X := ((TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth)/2.0) - ProcessWaferResults[CurrentWaferIndex].WaferCoM[0]) * VisionRecipe.Pixels.X_mm;
					WaferResults[CurrentWaferIndex].WaferCavityOffset.Y := ((TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight)/2.0) - ProcessWaferResults[CurrentWaferIndex].WaferCoM[1]) * VisionRecipe.Pixels.Y_mm;
				END_IF
				
				EdgeDetectionStartPoint[0] := 0.0;
				EdgeDetectionStartPoint[1] := TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight)/2.0;
				EdgeDetectionStopPoint[0] := TO_REAL( ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth) - 1.0;
				EdgeDetectionStopPoint[1] := TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight)/2.0;;
				F_VN_MeasureEdgeDistanceExp(workingImage,
											ProcessWaferResults[CurrentWaferIndex].WidthInPixels,
											EdgeDetectionStartPoint,
											EdgeDetectionStopPoint,
											etcvnedgedirection.TCVN_ED_DARK_TO_LIGHT,
											128.0,
											20,
											TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight)/(2.0*11.0),
											1,
											FALSE,
											0.0,
											1,
											0.001,
											ETcVnEdgeDetectionAlgorithm.TCVN_EDA_INTERPOLATION,
											ProcessWaferResults[CurrentWaferIndex].WaferLeftEdgePoints,
											ProcessWaferResults[CurrentWaferIndex].WaferRightEdgePoints,
											UnusedContainer,
											hrPrev);
				F_VN_GetNumberOfElements(ProcessWaferResults[CurrentWaferIndex].WaferLeftEdgePoints, ProcessWaferResults[CurrentWaferIndex].WaferPointsCount, hrPrev);	
				
				WaferResults[CurrentWaferIndex].WaferWidth := TO_REAL(VisionRecipe.Pixels.Y_mm * ProcessWaferResults[CurrentWaferIndex].WidthInPixels);					
	
				hrPrev := F_VN_CreateContainer(WaferFitLine, ContainerType_Vector_TcVnPoint2_REAL, 2, hrPrev);
	
				FiducialSearchPoint[0] := 0;
				FiducialSearchPoint[1] := 0;
					
				hrPrev := F_VN_AddToContainerElements_TcVnPoint2_REAL(FiducialSearchPoint, WaferFitLine, hrPrev);
				
				FiducialSearchPoint[0] := 100;
				FiducialSearchPoint[1] := 0;
					
				hrPrev := F_VN_AddToContainerElements_TcVnPoint2_REAL(FiducialSearchPoint, WaferFitLine, hrPrev);
					
				hrPrev := F_VN_FitLine(WaferFitLine, WaferEdgeLine, hrPrev); 
				
				hrPrev := F_VN_FitLine(ProcessWaferResults[CurrentWaferIndex].WaferLeftEdgePoints, ImageEdgeLine, hrPrev); 
				
				hrPrev := F_VN_LineIntersectionPointAndAngle(ImageEdgeLine,
														WaferEdgeLine, 
														UnusedIntersection,
														WaferResults[CurrentWaferIndex].WaferAngleInDegrees,
														TRUE,
														hrPrev);												
					
											
				EdgeDetectionStartPoint[0] := ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth/2.0;;
				EdgeDetectionStartPoint[1] := 0.0;
				EdgeDetectionStopPoint[0] := TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth)/2.0;
				EdgeDetectionStopPoint[1] := TO_REAL( ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight) - 1.0;
				F_VN_MeasureEdgeDistanceExp(workingImage,
											ProcessWaferResults[CurrentWaferIndex].LengthInPixels,
											EdgeDetectionStartPoint,
											EdgeDetectionStopPoint,
											etcvnedgedirection.TCVN_ED_DARK_TO_LIGHT,
											128.0,
											20,
											TO_REAL(ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight)/(2.0*11.0),
											1,
											FALSE,
											0.0,
											1,
											0.001,
											ETcVnEdgeDetectionAlgorithm.TCVN_EDA_INTERPOLATION,
											ProcessWaferResults[CurrentWaferIndex].WaferTopEdgePoints,
											ProcessWaferResults[CurrentWaferIndex].WaferBottomEdgePoints,
											UnusedContainer,
											hrPrev);
											
											
				WaferResults[CurrentWaferIndex].WaferLength := TO_REAL(VisionRecipe.Pixels.X_mm * ProcessWaferResults[CurrentWaferIndex].LengthInPixels);										
				IF WaferResults[CurrentWaferIndex].WaferLength < VisionRecipe.Wafer.MaxWaferLengthLimit
					AND WaferResults[CurrentWaferIndex].WaferLength > VisionRecipe.Wafer.MinWaferLengthLimit
					AND WaferResults[CurrentWaferIndex].WaferWidth < VisionRecipe.Wafer.MaxWaferWidthLimit
					AND WaferResults[CurrentWaferIndex].WaferWidth > VisionRecipe.Wafer.MinWaferWidthLimit THEN
					ProcessWaferResults[CurrentWaferIndex].WaferDimensionsOk := TRUE;
				ELSE
					ProcessWaferResults[CurrentWaferIndex].WaferDimensionsOk := FALSE;
				END_IF
				IF hrPrev <> 0 THEN
					hrPrev := 0;
				END_IF
			
{endregion}			

//take the corrected image
//copy region around the wafer cavity
// take an internal region inside the wafer dimensions
//find the max value within thr image
//step down in increments through the heights to establish amount of wafer in eaxch height region
//this assumes the highest point is good wafer
{region 'Find wafer delaminations'}
	
				hrPrev := F_VN_CopyImageRegion(CorrectedImage,
												ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
												ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
												TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
												TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm), 
												workingImage, 
												hrPrev);
				
				hrPrev := F_VN_CopyImageRegion(workingImage,
												TO_UDINT((ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth/2.0) - (VisionRecipe.Wafer.WaferStatsImageWidth/VisionRecipe.Pixels.X_mm)/2.0),
												TO_UDINT((ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight/2.0) - (VisionRecipe.Wafer.WaferStatsImageHeight/VisionRecipe.Pixels.Y_mm)/2.0),
												TO_UDINT(VisionRecipe.Wafer.WaferStatsImageWidth/VisionRecipe.Pixels.X_mm),
												TO_UDINT(VisionRecipe.Wafer.WaferStatsImageHeight/VisionRecipe.Pixels.Y_mm), 
												workingImage, 
												hrPrev);
				hrPrev := F_VN_PyramidDown(workingImage, workingImage, hrPrev);
				hrPrev := F_VN_PyramidDown(workingImage, workingImage, hrPrev);
				workingImage.GetImageInfo(ProcessWaferResults[CurrentWaferIndex].DelaminationImageInfo);
				F_VN_CreateImageAndSetPixels(LaminateMask, 
											ProcessWaferResults[CurrentWaferIndex].DelaminationImageInfo.nWidth, 
											ProcessWaferResults[CurrentWaferIndex].DelaminationImageInfo.nHeight,
											ETcVnElementType.TCVN_ET_UINT,
											1,
											Zero,	
											hrPrev); 
				F_VN_MaxPixelValue(workingImage, ProcessWaferResults[CurrentWaferIndex].MaxPixelValue,ProcessWaferResults[CurrentWaferIndex].MaxPixelLocation,hrPrev);
				FOR i := 0 TO VisionRecipe.Wafer.LaminateInspectionLayerNumber - 1 DO
					ProcessWaferResults[CurrentWaferIndex].DelaminationThreshold[i] := ProcessWaferResults[CurrentWaferIndex].MaxPixelValue[0] - ((i+1)*(VisionRecipe.Wafer.WaferLaminationHeight/VisionRecipe.Pixels.Z_mm));
					hrPrev := F_VN_Threshold(workingImage, 
										ThresholdImage,
										ProcessWaferResults[CurrentWaferIndex].DelaminationThreshold[i], 
										ThresholdMaxLevel,
										ETcVnThresholdType.TCVN_TT_BINARY,
										hrPrev);
					hrPrev := F_VN_SubtractImages(ThresholdImage,
												LaminateMask,
												ThresholdImage, 
												hrPrev);
					hrPrev := F_VN_CountNonZeroPixels(ThresholdImage, Pixels, hrPrev);
					WaferResults[CurrentWaferIndex].WaferLaminatePercentages[i] := TO_LREAL(Pixels)/(ProcessWaferResults[CurrentWaferIndex].DelaminationImageInfo.nHeight * ProcessWaferResults[CurrentWaferIndex].DelaminationImageInfo.nWidth)*100.0;
					F_VN_BitwiseOrImages(LaminateMask,ThresholdImage,LaminateMask, hrPrev);
				END_FOR
				ProcessWaferResults[CurrentWaferIndex].ThresholdSum := 0.0;
				FOR i := VisionRecipe.Wafer.LaminateStartLayer TO VisionRecipe.Wafer.LaminateStopLayer DO
					 ProcessWaferResults[CurrentWaferIndex].ThresholdSum := ProcessWaferResults[CurrentWaferIndex].ThresholdSum + WaferResults[CurrentWaferIndex].WaferLaminatePercentages[i];
				END_FOR
				IF ProcessWaferResults[CurrentWaferIndex].ThresholdSum < VisionRecipe.Wafer.MaxLaminateSum
								AND ProcessWaferResults[CurrentWaferIndex].ThresholdSum > VisionRecipe.Wafer.MinLaminateSum THEN
					ProcessWaferResults[CurrentWaferIndex].WaferDelamOk := TRUE;
				ELSE
					ProcessWaferResults[CurrentWaferIndex].WaferDelamOk := FALSE;
				END_IF

{endregion}
		
//take the corrected image
//copy region around the wafer cavity
//take the wafer and find the mean and standard deviation of height map
{region 'Extract wafer height mean and standard deviation data'}
				hrPrev := F_VN_CopyImageRegion(CorrectedImage,
												ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
												ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
												TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
												TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm), 
												workingImage, 
												hrPrev);
				
				hrPrev := F_VN_CopyImageRegion(workingImage,
												TO_UDINT((ProcessWaferResults[CurrentWaferIndex].ImageInfo.nWidth/2.0) - (VisionRecipe.Wafer.WaferStatsImageWidth/VisionRecipe.Pixels.X_mm)/2.0),
												TO_UDINT((ProcessWaferResults[CurrentWaferIndex].ImageInfo.nHeight/2.0) - (VisionRecipe.Wafer.WaferStatsImageHeight/VisionRecipe.Pixels.Y_mm)/2.0),
												TO_UDINT(VisionRecipe.Wafer.WaferStatsImageWidth/VisionRecipe.Pixels.X_mm),
												TO_UDINT(VisionRecipe.Wafer.WaferStatsImageHeight/VisionRecipe.Pixels.Y_mm), 
												workingImage, 
												hrPrev);
				
				hrPrev := F_VN_ConvertElementType(workingImage, workingImage, etcvnelementtype.TCVN_ET_LREAL, hrPrev);
				
				hrPrev := F_VN_MultiplyImageWithScalar(visionRecipe.Pixels.Z_mm,workingImage, workingImage, hrPrev);
				
				hrPrev := F_VN_ImageAverageStdDev(workingImage, ProcessWaferResults[CurrentWaferIndex].WaferMeanHeight, ProcessWaferResults[CurrentWaferIndex].WaferSD, hrPrev);
				
				WaferResults[CurrentWaferIndex].SDWaferHeight := ProcessWaferResults[CurrentWaferIndex].WaferSD[0];
				WaferResults[CurrentWaferIndex].MeanWaferHeight := ProcessWaferResults[CurrentWaferIndex].WaferMeanHeight[0] - (ProcessWaferResults[CurrentWaferIndex].BottomMouldLevel - visionRecipe.Cavities.EmptyCavityDepth/VisionRecipe.Pixels.Z_mm)*VisionRecipe.Pixels.Z_mm;
				
//Calculate rejection based on the various criteria, wafer height, width, height, fill, delamination				
				IF  WaferResults[CurrentWaferIndex].MeanWaferHeight < VisionRecipe.Wafer.MaxWaferMeanHeightLimit
								AND WaferResults[CurrentWaferIndex].MeanWaferHeight > VisionRecipe.Wafer.MinWaferMeanHeightLimit
								AND WaferResults[CurrentWaferIndex].SDWaferHeight < VisionRecipe.Wafer.MaxWaferSDHeightLimit
								AND WaferResults[CurrentWaferIndex].SDWaferHeight > VisionRecipe.Wafer.MinWaferSDHeightLimit THEN
					ProcessWaferResults[CurrentWaferIndex].WaferHeightOK := TRUE;
				ELSE
					ProcessWaferResults[CurrentWaferIndex].WaferHeightOK := FALSE;
				END_IF

				WaferResults[CurrentWaferIndex].WaferStatus := ProcessWaferResults[CurrentWaferIndex].WaferPercentageOK 
															AND ProcessWaferResults[CurrentWaferIndex].WaferHeightOK
															AND ProcessWaferResults[CurrentWaferIndex].WaferDelamOk
															AND ProcessWaferResults[CurrentWaferIndex].WaferDimensionsOk;
			END_IF
			hrPrev := 0;
			CurrentWaferIndex := CurrentWaferIndex + 1;							
{endregion}	

		END_FOR
		CurrentBarIndex := CurrentBarIndex + 1;
	END_FOR
END_FOR							

WaferTiming(START := FALSE, RESET := FALSE);
WaferTiming(START := FALSE, RESET := TRUE);	
	
{endregion}

//Takes the calibrated filtered image
//convert to RBG
//iterate over all wafers in array and apply a green or red box to indicate pass fail on the individual wafers
{region "Present Results generates an image to show wafer based pass/fail results"}

DisplayTiming(START := FALSE, RESET := FALSE);
DisplayTiming(START := TRUE, RESET := FALSE);	
	
hrPrev := F_VN_CopyImage(medianFilteredImage, workingDisplayImage, hrPrev);
hrPrev := F_VN_SubtractScalarFromImage(workingDisplayImage, ProcessSKUDiagResult.MouldMinLevel,workingDisplayImage, hrPrev);
hrPrev := F_VN_MultiplyImageWithScalar( (MaxValueIn8Bit / (ProcessSKUDiagResult.MouldMaxLevel-ProcessSKUDiagResult.MouldMinLevel)),workingDisplayImage, workingDisplayImage, hrPrev);
hrPrev := F_VN_ConvertElementType(workingDisplayImage, workingDisplayImage, etcvnelementtype.TCVN_ET_USINT, hrPrev);
hrPrev := ColourSpaceConversion.Transform(workingDisplayImage, workingDisplayImage, ETcVnColorSpaceTransform.TCVN_CST_GRAY_TO_RGB, hrPrev);

CurrentWaferIndex := 0;	
FOR Row := 0 TO ((VisionRecipe.SKU.NumberOfBars/VisionRecipe.SKU.NumberBarsPerRow) - 1) DO
	FOR Column := 0 TO (VisionRecipe.SKU.NumberBarsPerRow - 1) DO
		FOR Cavity := 0 TO (VisionRecipe.SKU.WafersPerBar - 1) DO
			IF WaferResults[CurrentWaferIndex].WaferStatus THEN
				hrPrev := F_VN_DrawRectangle(ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
											ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
											ProcessWaferResults[CurrentWaferIndex].WaferXPosition + TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
											ProcessWaferResults[CurrentWaferIndex].WaferYPosition + TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm),
 											workingDisplayImage,
											Green,
											2,
											hrPrev);
			ELSE
				hrPrev := F_VN_DrawRectangle(ProcessWaferResults[CurrentWaferIndex].WaferXPosition,
											ProcessWaferResults[CurrentWaferIndex].WaferYPosition,
											ProcessWaferResults[CurrentWaferIndex].WaferXPosition + TO_UDINT(VisionRecipe.Cavities.CavityWidth/VisionRecipe.Pixels.X_mm),
											ProcessWaferResults[CurrentWaferIndex].WaferYPosition + TO_UDINT(VisionRecipe.Cavities.CavityLength/VisionRecipe.Pixels.Y_mm),
 											workingDisplayImage,
											Red,
											2,
											hrPrev);
			END_IF
			CurrentWaferIndex := CurrentWaferIndex + 1;
		END_FOR
	END_FOR
END_FOR
	
hrPrev := F_VN_CopyIntoDisplayableImage(workingDisplayImage, FinalImage, hrPrev);	

DisplayTiming(START := FALSE, RESET := FALSE);
DisplayTiming(START := FALSE, RESET := TRUE);	
{endregion}	

{region "CleanUp, releases the image contianers to prevent memory leaks"}	
	
FW_SafeRelease(ADR(workingImage));
FW_SafeRelease(ADR(workingImage));
FW_SafeRelease(ADR(CorrectedImage));
FW_SafeRelease(ADR(ThresholdImage));
FW_SafeRelease(ADR(ThresholdInvImage));
FW_SafeRelease(ADR(Lvl1Image));
FW_SafeRelease(ADR(Lvl2Image));

FW_SafeRelease(ADR(medianFilteredImage));
FW_SafeRelease(ADR(ThresholdHighImage));
FW_SafeRelease(ADR(ThresholdLowImage));

Result := ProcessVisionResult(hrPrev);
Process := hrPrev;

TotalTiming(START := FALSE, RESET := FALSE);
TotalTiming(START := FALSE, RESET := TRUE);
{endregion}
```

---

## 5. Communications & Fieldbus Interfaces

### 5.1 EtherNet/IP Data Decoder (`DecodeRockwellData`)
Decodes the control byte received from the Rockwell ControlLogix 5580 line PLC into standard PackML commands.

```iecst
FUNCTION_BLOCK DecodeRockwellData
VAR_INPUT
    Name : STRING;
    SysCtrlByte : BYTE;
END_VAR
VAR_IN_OUT
    SysCtrl : SystemControl;
END_VAR

// Bit unpack from Rockwell Byte
SysCtrl.Run       := SysCtrlByte.0;
SysCtrl.Stop      := SysCtrlByte.1;
SysCtrl.Reset     := SysCtrlByte.2;
SysCtrl.Shutdown  := SysCtrlByte.3;
SysCtrl.HeartBeat := SysCtrlByte.7;
```

### 5.2 EtherNet/IP Data Encoder (`EncodeBeckhoffData`)
Encodes the Beckhoff inspection results and system status back into the 720-byte payload sent cyclically over EtherNet/IP to the line PLC for airknife reject triggering.

```iecst
FUNCTION_BLOCK EncodeBeckhoffData
VAR_INPUT
    Name : STRING;
    SysStatus : SystemStatus;
    WaferResults : ARRAY[0..199] OF WaferResultStruct;
END_VAR
VAR_OUTPUT
    SysStatusBytes : ARRAY[0..29] OF BYTE;
    WaferResultBytes : ARRAY[0..719] OF BYTE;
END_VAR
VAR
    i : INT;
    byteOffset : INT;
END_VAR

// 1. Pack System Status (Mode, State, Faults, Heartbeat)
SysStatusBytes[0].0 := SysStatus.Ready;
SysStatusBytes[0].1 := SysStatus.Running;
SysStatusBytes[0].2 := SysStatus.Faulted;
SysStatusBytes[0].3 := SysStatus.HeartBeatAck;
SysStatusBytes[1]   := TO_BYTE(SysStatus.CurrentState);
SysStatusBytes[2]   := TO_BYTE(SysStatus.CurrentMode);

// 2. Pack Individual Wafer Results (4 bytes per wafer cavity)
// Byte 0: Bit 0 = Pass, Bit 1 = Missing, Bit 2 = Height Fault, Bit 3 = Misplaced
// Byte 1: Cavity Row Index
// Byte 2: Cavity Column Index
// Byte 3: Average Height (scaled)
FOR i := 0 TO 199 DO
    byteOffset := i * 4;
    IF byteOffset + 3 <= 719 THEN
        WaferResultBytes[byteOffset + 0].0 := WaferResults[i].Pass;
        WaferResultBytes[byteOffset + 0].1 := WaferResults[i].MissingWafer;
        WaferResultBytes[byteOffset + 0].2 := WaferResults[i].HeightFault;
        WaferResultBytes[byteOffset + 0].3 := WaferResults[i].Misplaced;
        WaferResultBytes[byteOffset + 1]   := TO_BYTE(WaferResults[i].RowIndex);
        WaferResultBytes[byteOffset + 2]   := TO_BYTE(WaferResults[i].ColumnIndex);
        WaferResultBytes[byteOffset + 3]   := TO_BYTE(WaferResults[i].AvgHeight * 10.0);
    END_IF
END_FOR
```

---

## 6. Key Data Types & Structure Definitions (`TcDUT`)

### 6.1 `WaferResultStruct.TcDUT`
```iecst
TYPE WaferResultStruct :
STRUCT
    Pass : BOOL;                  // Overall Pass/Fail decision
    MissingWafer : BOOL;          // TRUE if no wafer detected in cavity
    Misplaced : BOOL;             // TRUE if wafer orientation/offset exceeds limits
    HeightFault : BOOL;           // TRUE if wafer height outside thickness bounds
    DoubleWafer : BOOL;           // TRUE if sticker / double wafer detected
    DelaminationFault : BOOL;     // TRUE if delamination detected
    AvgHeight : REAL;             // Measured mean height in mm relative to cavity base
    FillPercentage : REAL;        // Percentage of cavity area occupied by wafer
    RowIndex : USINT;             // Physical row inside mould (e.g. 1..6)
    ColumnIndex : USINT;          // Physical column/finger index (e.g. 1..24)
    WaferXPosition : UDINT;       // Detected pixel X coordinate
    WaferYPosition : UDINT;       // Detected pixel Y coordinate
END_STRUCT
END_TYPE
```

### 6.2 `SystemControl.TcDUT`
```iecst
TYPE SystemControl :
STRUCT
    Run : BOOL;
    Stop : BOOL;
    Reset : BOOL;
    Shutdown : BOOL;
    HeartBeat : BOOL;
    RecipeID : INT;
    ClearFaults : BOOL;
END_STRUCT
END_TYPE
```

### 6.3 `SystemStatus.TcDUT`
```iecst
TYPE SystemStatus :
STRUCT
    Ready : BOOL;
    Running : BOOL;
    Faulted : BOOL;
    HeartBeatAck : BOOL;
    CurrentState : INT;          // PackML state (1=Clearing, 2=Stopped, 3=Starting, 4=Idle, etc.)
    CurrentMode : INT;           // 1=Production, 2=Maintenance, 3=Manual
    InspectionActive : BOOL;
    TotalInspectedCount : UDINT;
    TotalRejectedCount : UDINT;
END_STRUCT
END_TYPE
```

### 6.4 `WaferDetectionVisionRecipe.TcDUT`
```iecst
TYPE WaferDetectionVisionRecipe :
STRUCT
    Pixels : PixelCalibration;
    VisionProcess : VisionProcessSettings;
    Cavities : CavityRecipe;
    Mould : MouldRecipe;
    InspectionLimits : WaferInspectionLimits;
END_STRUCT
END_TYPE
```

### 6.5 `PixelCalibration.TcDUT`
```iecst
TYPE PixelCalibration :
STRUCT
    SickRulerScale : LREAL := 0.00390625;   // 16-bit to float conversion
    SickRulerOffset : LREAL := 0.0;
    X_mm : LREAL := 0.370;                  // Horizontal mm per pixel
    Y_mm : LREAL := 0.200;                  // Conveyor travel mm per line
    Z_mm : LREAL := 0.010;                  // Depth resolution mm per count
END_STRUCT
END_TYPE
```

---

## 7. Critical Engineer Doubts & Gotchas

1. **Memory Leak Risk in `Process`**:
   The `Process` method allocates multiple TwinCAT Vision containers via `F_VN_CreateImage` and API calls (`workingImage`, `medianFilteredImage`, `CorrectedImage`, `ThresholdImage`, etc.). If an early return occurs (e.g. `IF NOT ValidRecipe THEN RETURN;`), the cleanup section at the end is bypassed, leading to rapid Beckhoff IPC memory exhaustion.
   *Fix Recommendation:* Enforce a single-exit pattern where cleanup is guaranteed via an exit action or `FINALLY` block.

2. **Global Coordinates vs Local ROIs**:
   Notice in Stage 6 that wafer cavities are iterated using bounding box offsets added to the detected mould origin (`FiducialResult`). If the fiducial search fails or returns a false lock due to chocolate crumbs, all cavity ROIs shift, causing false rejects across all 48 fingers.

3. **Nozzle Pitch vs Array Index**:
   The Beckhoff code outputs a 200-element array. The physical line has a 48-station SMC airknife manifold at 19.2 mm pitch. The mapping from wafer index (0..23 for 2F, 0..11 for 4F) to airknife nozzles (1..48) is handled in software. Ensure that the active SKU correctly maps to paired, quad, or triplet nozzles as documented in `DELTA_COMPARISON.md`.
