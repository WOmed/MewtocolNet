# MewtocolNet
A Mewtocol protocol library to interface with Panasonic PLCs over TCP/Serial.

## Disclaimer 
This library is not an official panasonic product nor does panasonic provide financial support or limitations in any form. 
This software was written by WOLF Medizintechnik GmbH (@WOmed/dev).

# Support

> This library was only tested with a few PLCs, other types that support the Panasonic Mewtocol protocol might work. 
> Use at your own risk, others might follow with community feedback

|PLC Type|Supported|Tested|
|--------|---------|------|
FPX C14  |✔        |✔   |
FPX C30  |✔        |✔   |
FPX-H C14|✔        |✔   |
FPX-H C30|✔        |✔   |
FP Sigma |✔        |❌  |

Where is the RS232/Serial support

> Support for the serial protocol will be added soon, feel free to contribute

# Protocol description

Panasonic has published a [protocol definition](https://mediap.industry.panasonic.eu/assets/custom-upload/Factory%20&%20Automation/PLC/Manuals/mn_all_plcs_mewtocol_user_pidsx_en.pdf) on their site.
Refer to this site if you want to see the general functionality or add / report missing features.

> This library is at the time not feature complete, but all essential features are provided

# Usage

See [More examples](/Examples) here

## Connecting to a PLC 

Connecting to a PLC is as simple as 

```C#
 //attaching a logger
Logger.LogLevel = LogLevel.Verbose;
Logger.OnNewLogMessage((date, msg) => {
    Console.WriteLine($"{date.ToString("HH:mm:ss")} {msg}");
});

//setting up a new PLC interface
MewtocolInterface interf = new MewtocolInterface("10.237.191.3");

await interf.ConnectAsync();
```

You can also use the callbacks of the `ConnectAsync()` method to do something after the initial connection establishment.

```C#
await interf.ConnectAsync(
    //PLC connected
    (plc) => {
        if(plcinf.OperationMode.RunMode)
            Console.WriteLine("PLC is in RUN");
    },
    //Connection failed
    () => {
        Console.WriteLine("PLC failed to connect");
    }
);
```
## Reading data registers / contacts

- Create a new class that inherits from `RegisterCollectionBase`

```C#
public class TestRegisters : RegisterCollectionBase {

    //corresponds to a R100 boolean register in the PLC
    [Register(100, RegisterType.R)]
    public bool TestBool1 { get; private set; }

    //corresponds to a XD input of the PLC
    [Register(RegisterType.X, SpecialAddress.D)]
    public bool TestBoolInputXD { get; private set; } 

    //corresponds to a DT7012 - DT7013 as a 32bit time value that gets parsed as a timespan (TIME)
    //the smallest value to communicate to the PLC is 10ms
    [Register(7012)]
    public TimeSpan TestTime { get; private set; }  
    
    //corresponds to a DT1101 - DT1104 string register in the PLC with (STRING[4])
    [Register(1101, 4)]
    public string TestString1 { get; private set; }

}
```

- Connect to the PLC and attach the register collection and logger
- attach an automatic poller by chaining `.WithPoller()` after the register attachment

```C#
//attaching a logger
Logger.LogLevel = LogLevel.Verbose;
Logger.OnNewLogMessage((date, msg) => {
    Console.WriteLine($"{date.ToString("HH:mm:ss")} {msg}");
});

//setting up a new PLC interface and register collection
MewtocolInterface interf = new MewtocolInterface("192.168.115.3");
TestRegisters registers = new TestRegisters();

//attaching the register collection and an automatic poller
interf.WithRegisterCollection(registers).WithPoller();

await interf.ConnectAsync(
    (plcinf) => {
        //reading a value from the register collection
        Console.WriteLine($"Time Value is: {registers.TestTime}");
    }
);
```
- Your properties are getting automatically updated after the initial connection

> Note! this is not your only option to read registers, see here

## Writing data registers / contacts

### Synchronous

Sets the register without feedback if it was set

```C#
//inverts the boolean register
interf.SetRegister(nameof(registers.TestBool1), !registers.TestBool1);

//set the current second to the PLCs TIME register
interf.SetRegister(nameof(registers.TestTime), TimeSpan.FromSeconds(DateTime.Now.Second));

 //writes 'Test' to the PLCs string register
interf.SetRegister(nameof(registers.TestString1), "Test");
```

You can also set a register by calling its name directly (Must be either in an attached register collection or added to the list manually)

### Asynchronous

Sets the register waiting for the PLC to confirm it was set

```C#
//inverts the boolean register
interf.SetRegisterAsync(nameof(registers.TestBool1), !registers.TestBool1);
```
