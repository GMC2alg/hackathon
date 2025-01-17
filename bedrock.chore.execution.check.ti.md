#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.chore.execution.check', 'pLogOutput', pLogOutput,
        'pStrictErrorHandling', pStrictErrorHandling,
        'pMonthDays', '', 
        'pWeekDays', '',
        'pDelim', '&',
        'pStartTime', 0, 'pEndTime', 24
    );
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~## 
################################################################################################# 

#Region @DOC
# Description:
# This TI was created to overcome the limited scheduling options in chores. In order to use this TI it has to be the 1st TI in the chore.
# As an example, if you need to run a chore every Monday & Wednesday you would schedule it to run EVERY day but set the pWeekdays parameter to Mon & Wed.
# The chore would then kick off every day but this TI will perform a ProcessExitByChoreQuit function on all days NOT mentioned in pWeekdays.

# Alternative setup: instead of adding this TI as the 1st TI in the chore, you could also use ExecuteProcess to call it, 
# from the top of the Prolog of your first TI process. This is definitely easier than hardcoding process call parameters in the chore dialog.

# Use case: For productive systems.
# 1. A chore should run every 30 minutes between 8am & 8pm on weekdays. Schedule chore for every 30 minutes and include this process 1st in chore with parameters pWeekDays=MON&TUE&WED&THU&FRI pStartTime=8 pEndTime=20.
# 2. A chore should run only on 1st calendar day of each month. Schedule chore for daily execution and include this process 1st in chore with parameters pMonthDays=1.

# Note:
# * This process will quit a chore if any time-bound, weekday-bound or date-bound conditions which define when the chore should NOT run are met.
# * Only the parameter(s) needed should be specified.
# * Only scheduled executions will be quit outside the parameters. The checks are bypassed if a chore is manually executed by a user. This is done by checking the TM1User function.
# * Time conditions are checked using these parameters in the following order of priority.
#   1. pMonthDays : Days in month when chore is allowed to run. Enter delimited list of days e.g. 1&2&30&31 (blank = no restriction on allowed days of month).
#   2. pWeekDays : Days in week when chore is allowed to run Enter delimited list of weekdays e.g. MON&FRI (blank = no restriction on allowed weekdays).
#   3. pStartTime & pEndTime : Time of day when chore is allowed to run e.g. pStartTime=7, pEndTime=22 execution will be allowed between 7AM & 10PM ( blank = no time-bound restrictions).
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName   = GetProcessName();
cUserName       = TM1User();
cTimeStamp      = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt      = NumberToString( INT( RAND( ) * 1000 ));
cTempSub        = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cStartTime      = NumberToString( pStartTime );
cEndTime        = NumberToString( pEndTime );
cMsgErrorLevel  = 'ERROR';
cMsgErrorContent= 'User:%cUserName% Process:%cThisProcName% Message: %sMsg%';
cLogInfo        = 'User:%cUserName% Process:%cThisProcName% run to check if chore should run with parameters pMonthDays:%pMonthDays%, pWeekDays:%pWeekDays%, pDelim:%pDelim%, pStartTime:%cStartTime%, pEndTime:%cEndTime%.' ; 
nErrors         = 0;
sMsg            = '';

## LogOutput parameters
IF( pLogOutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
EndIf;

### Check params
If( pDelim @= '' );
    pDelim = '&';
Else;
    pDelim = SubSt( pDelim, 1, 1 );
EndIf;

If( pMonthDays @= 'ALL' );
    pMonthDays = '';
EndIf;
If( pMonthDays @<> '' );
    If( SubSt( pMonthDays, Long( pMonthDays ), 1 ) @<> pDelim );
        pMonthDays = pMonthDays | pDelim;
    EndIf;
EndIf;

If( pWeekDays @= 'ALL' );
    pWeekDays = '';
EndIf;
If( pWeekDays @<> '' );
    pWeekDays = Upper( pWeekDays );
    If( SubSt( pWeekDays, Long( pWeekDays ), 1 ) @<> pDelim );
        pWeekDays = pWeekDays | pDelim;
    EndIf;
EndIf;

If( pStartTime <= 0 % pStartTime > 24 );
    pStartTime = 0;
Else;
    pStartTime = Round(pStartTime);
EndIf;
sStartTime = NumberToString( pStartTime );

If( pEndTime <= 0 % pEndTime > 24 );
    pEndTime = 24;
Else;
    pEndTime = Round(pEndTime);
EndIf;

If( pEndTime < pStartTime );
    pEndTime = pStartTime;
EndIf;
sEndTime = NumberToString( pEndTime );

### Initialize quit Boolean
bQuit = 0;

### Check the user
If( DIMIX( '}Clients', cUserName ) > 0 );
    If( pLogOutput >= 1 );
        sMsg = 'This chore will NOT quit since executed by a user.';
        LogOutput( 'INFO', Expand( cMsgErrorContent ) );
    EndIf;
Else;
    
    ### Check the day of the month
    If( pMonthDays @<> '' );
        sDayInMonth = TimSt(Now, '\d');
        If( Scan( sDayInMonth | pDelim, pMonthDays ) = 0 & Scan( sDayInMonth |' '| pDelim, pMonthDays ) = 0 );
            # could not find the day in the list of acceptable days
            bQuit = 1;
            sMsg = Expand('Bedrock debug %cThisProcName%: chore will quit. Could not find today %sDayInMonth% in list of acceptable days %pMonthDays%');
            IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
        Else;
            sMsg = Expand('Bedrock debug %cThisProcName%: today %sDayInMonth% found in list of acceptable days %pMonthDays%');
            IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
        EndIF;
    EndIf;

    ### Check the day of the week
    If( pWeekDays @<> '' );
        # support for UseExcelSerialDate=T in TM1s.cfg
        nDayIndex = Mod( DayNo( Today ) + Dayno( '1960-01-01' ) / 21916 - 2, 7 );
        sWeekday = '';
        If( nDayIndex = 0 );
            sWeekday = 'SUN';
        ElseIf( nDayIndex = 1 );
            sWeekday = 'MON';
        ElseIf( nDayIndex = 2 );
            sWeekday = 'TUE';
        ElseIf( nDayIndex = 3 );
            sWeekday = 'WED';
        ElseIf( nDayIndex = 4 );
            sWeekday = 'THU';
        ElseIf( nDayIndex = 5 );
            sWeekday = 'FRI';
        ElseIf( nDayIndex = 6 );
            sWeekday = 'SAT';
        EndIf;
        If( Scan( sWeekday | pDelim, pWeekDays ) = 0 & Scan( sWeekday |' '| pDelim, pWeekDays ) = 0 );
            # could not find the day in the list of acceptable days
            bQuit = 1;
            pWeekDays = Delet( pWeekDays, Long( pWeekDays ), 1 );
            sMsg = Expand('Bedrock debug %cThisProcName%: chore will quit. Could not find today %sWeekday% in list of acceptable days %pWeekDays%');
            IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
        Else;
            pWeekDays = Delet( pWeekDays, Long( pWeekDays ), 1 );
            sMsg = Expand('Bedrock debug %cThisProcName%: today %sWeekday% found in list of acceptable days %pWeekDays%');
            IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
        EndIF;
    EndIf;
    
    ### Check the time of day
    sMinute = TimSt(Now, '\h:\i');
    vTimeNow = StringToNumber(SubSt(sMinute, 1, 2));
    If( pStartTime = 0 & pEndTime = 24 );
        # no time exclusion parameters are set
    ElseIf( vTimeNow < pStartTime % vTimeNow >= pEndTime );
        # we are in the exclusion zone do not execute chore
        bQuit = 1;
        sMsg = Expand('Bedrock debug %cThisProcName%: chore will quit. current time %sMinute% is outside the defined execution time from %sStartTime%:00 to %sEndTime%:00');
        IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
    Else;
        # we are not in the exclusion zone, proceed as normal 
        sMsg = Expand('Bedrock debug %cThisProcName%: current time %sMinute% is within the defined execution time from %sStartTime%:00 to %sEndTime%:00');
        IF( pLogOutput = 1 ); LogOutput( 'INFO', sMsg ); EndIf;
    EndIF;

EndIf;

### Quit chore if quit conditions met
If( bQuit = 1 );
    sMsg = Expand('Bedrock debug %cThisProcName%: terminated the chore for the reasons stated above.');
    If( pLogOutput = 1 ); LogOutput( 'INFO' , Expand( cMsgErrorContent ) ); EndIf;
    nProcessReturnCode = ProcessExitByChoreQuit();
    sProcessReturnCode = 'ProcessExitByChoreQuit';
    ChoreQuit;
Else;
    ### Return Code
    sProcessAction      = Expand('Bedrock debug %cThisProcName%: validated the chore to run as normal.');
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****
#endregion
#region Data

#****Begin: Generated Statements***
#****End: Generated Statements****
#endregion
#region Epilog

#****Begin: Generated Statements***
#****End: Generated Statements****

### Return code & final error message handling
If( nErrors > 0 );
    sMessage = 'the process incurred at least 1 error. Please see above lines in this file for more details.';
    nProcessReturnCode = 0;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sProcessReturnCode = Expand( '%sProcessReturnCode% Process:%cThisProcName% completed with errors. Check tm1server.log for details.' );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    EndIf;
Else;
    sProcessAction = Expand( 'Process:%cThisProcName% completed normally' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogOutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion