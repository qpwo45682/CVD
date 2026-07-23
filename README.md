# CVD


// ============================================================================
// Cumulative Volume Delta (CVD) ? Three Session Version
// ============================================================================
//
// Session 1: Day Session    08:45 ~ 13:45
// Session 2: Night Session  15:00 ~ 05:00
// Session 3: Full Session   15:00 ~ Next Day 13:45 (Night + Day continuous)
//
// ============================================================================

Inputs:
    iDayStartTime(845),
    iDayEndTime(1345),

    iNightStartTime(1500),
    iNightEndTime(500),

    // Full Session: 15:00 ~ Next Day 13:45
    iFullStartTime(1500),
    iFullEndTime(1345),

    // 1 = Use Ticks
    // 2 = Use Volume
    iQtyMode(1),

    // True: Carry previous direction when price is flat
    // False: Delta = 0 when price is flat
    iCarryDirectionOnFlat(True),

    // Select which session CVD to display
    // 1 = Day Session
    // 2 = Night Session
    // 3 = Full Session (15:00 ~ Next Day 13:45)
    iDisplayMode(3);


Vars:
    // --- Day Session ---
    vInDay(False),
    vPrevInDay(False),
    vNewDaySession(False),
    vCVD_Day(0),
    vCVDOpen_Day(0),
    vBarDelta_Day(0),
    vLastDirection_Day(0),

    // --- Night Session ---
    vInNight(False),
    vPrevInNight(False),
    vNewNightSession(False),
    vCVD_Night(0),
    vCVDOpen_Night(0),
    vBarDelta_Night(0),
    vLastDirection_Night(0),

    // --- Full Session (15:00 ~ Next Day 13:45) ---
    vInFull(False),
    vNewFullSession(False),
    vCVD_Full(0),
    vCVDOpen_Full(0),
    vBarDelta_Full(0),
    vLastDirection_Full(0),

    // --- Shared Variables ---
    vQty(0),
    vDirection(0),

    // --- Display Variables ---
    vPlotCVD(0),
    vPlotDelta(0),
    vInActiveSession(False);


// ============================================================================
// Determine which session the current bar belongs to
// ============================================================================

vInDay = False;
vInNight = False;
vInFull = False;


// Day Session: 08:45 ~ 13:45
if Time >= iDayStartTime and
   Time <= iDayEndTime then begin

    vInDay = True;

end;


// Night Session: 15:00 ~ 05:00 (crosses midnight)
if Time >= iNightStartTime or
   Time <= iNightEndTime then begin

    vInNight = True;

end;


// Full Session: 15:00 ~ Next Day 13:45 (crosses midnight)
// Covers 15:00 ~ 23:59 and 00:00 ~ 13:45
if Time >= iFullStartTime or
   Time <= iFullEndTime then begin

    vInFull = True;

end;


// ============================================================================
// Determine which session the previous bar belongs to
// ============================================================================

vPrevInDay = False;
vPrevInNight = False;


if CurrentBar > 1 then begin

    // Check if previous bar was in Day Session
    if Time[1] >= iDayStartTime and
       Time[1] <= iDayEndTime then begin

        vPrevInDay = True;

    end;

    // Check if previous bar was in Night Session
    if Time[1] >= iNightStartTime or
       Time[1] <= iNightEndTime then begin

        vPrevInNight = True;

    end;

end;


// ============================================================================
// Detect new session start
// ============================================================================

vNewDaySession = False;
vNewNightSession = False;
vNewFullSession = False;


if CurrentBar = 1 then begin

    // First bar: mark as new session if currently in session
    if vInDay then
        vNewDaySession = True;

    if vInNight then
        vNewNightSession = True;

    if vInFull then
        vNewFullSession = True;

end
else begin

    // Day Session start: previous bar was not in day session, current bar enters
    if vInDay and vPrevInDay = False then begin

        vNewDaySession = True;

    end;

    // Night Session start: previous bar was not in night session, current bar enters
    if vInNight and vPrevInNight = False then begin

        vNewNightSession = True;

    end;

    // ----------------------------------------------------------------
    // Full Session start detection (KEY FIX)
    // ----------------------------------------------------------------
    // Cannot rely on "previous bar not in full session" because
    // the full session covers 15:00~13:45 (almost 23 hours).
    // If there is no data during 13:46~14:59 gap, both previous
    // and current bars would be in full session, so the reset
    // would never trigger.
    //
    // Solution: Detect when current bar crosses into 15:00 territory
    // while previous bar was in the 13:45 or earlier zone.
    // ----------------------------------------------------------------

    // Method: Current time >= 15:00 AND previous bar time < 15:00
    // This catches the transition from day session (or gap) into night open
    if Time >= iFullStartTime and
       Time[1] < iFullStartTime then begin

        vNewFullSession = True;

    end;

end;


// ============================================================================
// Reset CVD at session start
// ============================================================================

// Reset Day Session CVD
if vNewDaySession then begin

    vCVD_Day = 0;
    vCVDOpen_Day = 0;
    vLastDirection_Day = 0;

end;

// Reset Night Session CVD
if vNewNightSession then begin

    vCVD_Night = 0;
    vCVDOpen_Night = 0;
    vLastDirection_Night = 0;

end;

// Reset Full Session CVD
if vNewFullSession then begin

    vCVD_Full = 0;
    vCVDOpen_Full = 0;
    vLastDirection_Full = 0;

end;


// ============================================================================
// Calculate quantity and direction (shared logic)
// ============================================================================

// Get trade quantity based on selected mode
if iQtyMode = 1 then begin

    vQty = Ticks;

end
else begin

    vQty = Volume;

end;

if vQty < 0 then
    vQty = 0;


// Determine price direction
// Close > Previous Close => Bullish
// Close < Previous Close => Bearish
// If equal, use Close vs Open as fallback
vDirection = 0;

if CurrentBar > 1 then begin

    if Close > Close[1] then begin

        vDirection = 1;

    end
    else if Close < Close[1] then begin

        vDirection = -1;

    end
    else begin

        // Close equals previous close, use intra-bar direction
        if Close > Open then begin

            vDirection = 1;

        end
        else if Close < Open then begin

            vDirection = -1;

        end
        else begin

            // Completely flat bar, direction depends on user setting
            vDirection = 0;

        end;

    end;

end;


// ============================================================================
// Day Session CVD accumulation
// ============================================================================

if vInDay then begin

    vCVDOpen_Day = vCVD_Day;

    Value1 = vDirection;

    // Handle flat bar: carry previous direction if enabled
    if Value1 = 0 then begin

        if iCarryDirectionOnFlat then
            Value1 = vLastDirection_Day;

    end;

    vBarDelta_Day = Value1 * vQty;
    vCVD_Day = vCVD_Day + vBarDelta_Day;

    // Update last known direction
    if Value1 <> 0 then
        vLastDirection_Day = Value1;

end;


// ============================================================================
// Night Session CVD accumulation
// ============================================================================

if vInNight then begin

    vCVDOpen_Night = vCVD_Night;

    Value2 = vDirection;

    // Handle flat bar: carry previous direction if enabled
    if Value2 = 0 then begin

        if iCarryDirectionOnFlat then
            Value2 = vLastDirection_Night;

    end;

    vBarDelta_Night = Value2 * vQty;
    vCVD_Night = vCVD_Night + vBarDelta_Night;

    // Update last known direction
    if Value2 <> 0 then
        vLastDirection_Night = Value2;

end;


// ============================================================================
// Full Session CVD accumulation (15:00 ~ Next Day 13:45)
// ============================================================================

if vInFull then begin

    vCVDOpen_Full = vCVD_Full;

    Value3 = vDirection;

    // Handle flat bar: carry previous direction if enabled
    if Value3 = 0 then begin

        if iCarryDirectionOnFlat then
            Value3 = vLastDirection_Full;

    end;

    vBarDelta_Full = Value3 * vQty;
    vCVD_Full = vCVD_Full + vBarDelta_Full;

    // Update last known direction
    if Value3 <> 0 then
        vLastDirection_Full = Value3;

end;


// ============================================================================
// Plot output based on selected display mode
// ============================================================================

vInActiveSession = False;
vPlotCVD = 0;
vPlotDelta = 0;


if iDisplayMode = 1 then begin

    // Display Day Session CVD
    vInActiveSession = vInDay;
    vPlotCVD = vCVD_Day;
    vPlotDelta = vBarDelta_Day;

end
else if iDisplayMode = 2 then begin

    // Display Night Session CVD
    vInActiveSession = vInNight;
    vPlotCVD = vCVD_Night;
    vPlotDelta = vBarDelta_Night;

end
else if iDisplayMode = 3 then begin

    // Display Full Session CVD (15:00 ~ Next Day 13:45)
    vInActiveSession = vInFull;
    vPlotCVD = vCVD_Full;
    vPlotDelta = vBarDelta_Full;

end;


// Draw CVD line with color based on current bar delta direction
if vInActiveSession then begin

    Plot1(vPlotCVD, "CVD");
    Plot2(0, "Zero");

    // Green (Cyan) for positive delta, Red for negative, Yellow for flat
    if vPlotDelta > 0 then
        SetPlotColor(1, Cyan)

    else if vPlotDelta < 0 then
        SetPlotColor(1, Red)

    else
        SetPlotColor(1, Yellow);

end
else begin

    // Outside active session: hide CVD line, keep zero line
    NoPlot(1);
    Plot2(0, "Zero");

end;
