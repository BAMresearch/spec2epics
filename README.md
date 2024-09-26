# Migrate IMS motor config to EPICS

Some helpful links:
- [EPICS IOC command reference](https://epics.anl.gov/EpicsDocumentation/AppDevManuals/AppDevGuide/3.12BookFiles/chapter2.html)
- [EPICS motor record reference](https://epics-modules.github.io/motor/motorRecord.html#Fields_alphabetical)
- [SPEC motor parameters config_adm reference](https://certif.com/spec_help/config_adm.html#motor-parameters)
- [MDrive Serial Programming Reference](https://novantaims.com/downloads/manuals/MCode.pdf#%5B%7B%22num%22%3A106%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22FitH%22%7D%2C1010%5D)

## Usage

Config files with imaginary network addresses can be found in the [testdata](testdata) subdir. Store your real files which where extracted from SPEC in subdir *cfg* out of the git tree (that subfolder is set to be ignored by GIT). For regenerating the IOC files, run the following (feel free to replace *testdata* by *cfg*):

    ./convert.py -c testdata/config_adm.txt -a testdata/spec_motor_aliases.md -l testdata/limits.txt -p ../synapps/support/motor-R7-3-1/modules/motorIms

The `-p` argument expects an existing path to the motorIms package source tree structure. The scripts searches for the *ims* binary and the *envPaths* file which should exist there after building it successfully. We used the [synApps package](https://github.com/EPICS-synApps) and our [modified assemble script](https://github.com/BAMresearch/EPICS-synApps-assemble) helps with building everything successfully on a Linux Ubuntu system.

Some help is shown as usual by:

    ./convert.py -h
 
It creates or updates files in the *generated* subfolder:
 
    generated/
    ├── moxa1.cmd
    ├── moxa1.substitutions
    ├── moxa2.cmd
    ├── moxa2.substitutions
    ├── moxa3.cmd
    ├── moxa3.substitutions
    ├── moxa4.cmd
    ├── moxa4.substitutions
    ├── moxa5.cmd
    ├── moxa5.substitutions
    ├── moxa6.cmd
    └── moxa6.substitutions
 
Start one IOC for the connected motors like this (in this case for *old[xy]sam*):
 
    ./generated/moxa4.cmd

## Translate SPEC motor settings to IOC motor record fields

### Additional info by Anja

Of the motors, the MDRIVE are the real, physical ones as far as I can see and MAC_MOT are aliases. You can ignore all TMCL motors, they are handled by the Trinamic controller (and the spec-config doesn't work for them anyway).

About the meaning of the motors, as far as I know them:

- s1bot, s1top should control the lower and upper blade of slot 1 respectively. Same goes for s1hr and s1hl for the horizontal. The other two slots (2 and 3) have the same motors, only starting with s2/s3.
- The aliases vg1 / hg1 etc. denote the vertical/horizontal opening of the associated slot pairs (s1bot + s1top, etc.) hp/vp stands analogously for horizontal/vertical position (of the slot pair).
- bsr and bsz are the real motors of the beam stop (rotation and vertical translation), aliases are bsh (horizontal) and bsv (vertical)
- detx, dety and detz control the position of the detector. Where detx is the distance between the sample and the detector, dety is the horizontal movement across the beam and detz is the vertical movement.
- dual should be for the movement of the X-ray sources, but I'm not so sure about that.

### Example: SPEC for *oldysam*

(See [config_adm file](testdata/config_adm.txt) for more.)

    # Motor      cntrl     steps sign  slew base  backl accel nada flags   mne  name
    MOT026 =  MDRIVE:3/33  34133   -1 51200  100 -34133  125    0 0x003  oldysam  oldysam
    MOTPAR:init_sequence = S1=2,0,1;S2=3,0,1
    MOTPAR:units = mm

- Field 2, Steps per unit (sign changes direction of motion): 34133
  - MRES = UREV/SREV (https://epics-modules.github.io/motor/motorRecord.html#Fields_res)
  - MRES = 1/34133 = 0.0000292972
- Field 5, Base rate (Hz): 100 (steps)
  - (base rate in steps per sec.) / (steps per unit) = (unit per sec.)
  - VBAS = 100 / 34133 = 0.002929716
  - (https://epics-modules.github.io/motor/motorRecord.html#Fields_motion)
- Field 4, Steady state rate (Hz): 51200
  - (Steady state rate in steps per sec.) / (steps per unit) = (unit per sec.)
  - VELO = 51200 / 34133 = 1.5
  - (https://epics-modules.github.io/motor/motorRecord.html#Fields_motion)
- Field 7, Acceleration time (msec): 125
  - ACCL = 0.125
  - (https://epics-modules.github.io/motor/motorRecord.html#Fields_motion)
- Field 6, Steps for backlash: -34133
  - (steps) / (steps per unit) = (distance in unit)
  - BDST = -34133 / 34133 = -1
  - **Note**: No extra values for backlash velocity and acceleration are set in SPEC
    therefore, use the values for forward operation
- Missing in SPEC config: the length of the axis, ie. movement limits set by *DHLM* and *DLLM*, what is allowed to be requested by *DVAL*.
  - (https://epics-modules.github.io/motor/motorRecord.html#Fields_limit)

Motor record:

        {   P,   N,       M,        DTYP,  PORT,       DESC, ADDR, EGU, DIR,         MRES,        VBAS, VELO,  ACCL, BDST,  BVEL,  BACC, DHLM, DLLM,               INIT }
        {IMS:, "x", "m$(N)", "asynMotor",  IMSX, "MDrive23",    0,  mm, Pos, 0.0000292972, 0.002929716,  1.5, 0.125,   -1,   1.5, 0.125,  100, -100, "S1=2,0,1;S2=3,0,1"}

### Motor limits in spec 

Motor limits (dial values) and offset can be queried in spec. 

    for (i = 0; i < MOTORS; i++) { 
      printf("Motor %s: address   %s; lower_limit %g; upper_limit %g; offset %g\n", 
      motor_name(i), motor_par(i, "device_id"), get_lim(i, -1), get_lim(i, 1), motor_par(i, "offset")); 
    }

Output is in [limits.txt](testdata/limits.txt)

- `get_lim(i, -1)` gets the lower limit (dial value)
  = DLLM
- `get_lim(i, -1)` gets the upper limit (dial value)
  = DHLM
- `motor_par(i, "offset")` gets the offset
  = OFF

## Current config files

### IOCsh script

Initially stored under `motor-R7-3-1/modules/motorIms/iocs/imsIOC/iocBoot/iocIms/maus.cmd`:

    #!../../bin/linux-x86_64/ims
    #
    # make sure the motors are in echo mode 2:
    # $ socat - TCP4:192.168.0.5:7434
    # XEM=2
    # ZEM=2
    # <CTRL-J>
    #
    # In serial comm. (above) print motor Z parameters with:
    # Z PR AL
    #
    # When the ioc is running, check availablePVs with `dbl`, for example:
    # epics> dbl
    # IMS:mxOffset
    # IMS:mxResolution
    # IMS:mzOffset
    # IMS:mzResolution
    # IMS:IOC:1:IMS_S
    # IMS:mxDirection
    # IMS:mzDirection
    # IMS:mx
    # IMS:mz

    < envPaths

    # change to the 'motor-R7-3-1/modules/motorIms/iocs/imsIOC' dir
    cd "${TOP}"

    # Register all support components
    dbLoadDatabase "dbd/ims.dbd"
    ims_registerRecordDeviceDriver pdbbase

    # Motors substitutions, customize this for your motor
    dbLoadTemplate "iocBoot/iocIms/maus.substitutions"

    # Configure asyn communication port, first
    # drvAsynIPPortConfigure(IOPortName, port, priority, disable auto-connect, no process EOS)
    drvAsynIPPortConfigure("moxa4", "192.168.0.5:7434", 0, 0, 0 )

    # Configure one controller per motor, each controller just has 1 axis
    # motorPortName, portName, deviceName, movingPollPeriod, idlePollPeriod
    ImsMDrivePlusCreateController("IMSZ", "moxa4", "Z", 200, 5000)
    ImsMDrivePlusCreateController("IMSX", "moxa4", "X", 200, 5000)

    # Optional: Enable tracing
    #asynSetTraceIOMask("IMS1", 0, 0)
    #asynSetTraceMask("IMS1", 0, 9)

    # motorUtil (allstop & alldone)
    #dbLoadRecords("$(MOTOR)/db/motorUtil.db", "P=ims:")

    # Initialize the IOC and start processing records
    iocInit()

    # motorUtil (allstop & alldone)
    #motorUtilInit("ims:")

### Substitutions file

Initial test settings (records) for motors *oldysam* and *oldzsam* stored under `iocBoot/iocIms/maus.substitutions` and referenced in the IOCsh script above:

    file "$(MOTOR)/db/basic_asyn_motor.db"
    {
    pattern
    {   P,   N,       M,        DTYP,  PORT,       DESC, ADDR, EGU, DIR, VELO,        VBAS, ACCL, BDST,  BVEL, BACC, DHLM, DLLM,         MRES, PREC, INIT, OFF}
    {IMS:, "x", "m$(N)", "asynMotor",  IMSX, "MDrive23",    0,  mm, Neg,   1.5, 0.002929716, 0.125,  -1,  1.5, 0.25,   52,  -52, 0.0000292972,    4,   "",   0}
    {IMS:, "z", "m$(N)", "asynMotor",  IMSZ, "MDrive23",    0,  mm, Pos, 1.875, 0.002929716, 0.175,   1,  1.5, 0.25,   16,  -88, 0.0000585944,    4,   "",   0}
    }

    file "$(TOP)/db/IMS_extra.db"
    {
        pattern
        {DEV, AREA, LOC, PORT, ADDR, TIMEOUT}
        {IMS,  IOC,   1, IMSX,    0,    1}
        {IMS,  IOC,   1, IMSZ,    0,    1}
    }


Updated values are derived from formulae above, limits as shown in the spec config.
