[![NuGet version](https://badge.fury.io/nu/Unosquare.Raspberry.IO.svg)](https://badge.fury.io/nu/Unosquare.Raspberry.IO)
[![Analytics](https://ga-beacon.appspot.com/UA-8535255-2/unosquare/raspberryio/)](https://github.com/igrigorik/ga-beacon)

# <img src="https://github.com/unosquare/raspberryio/raw/master/logos/raspberry-io.png" width="32" height="32"></img> RaspberryIO - *Pi's hardware access from Mono*
The Raspberry Pi's IO Functionality in an easy-to-use API for Mono/.NET/C#

*:star:Please star this project if you find it useful!*

## Features

This library enables developers to use the various Raspberry Pi's hardware modules
* ```Pi.Camera``` Provides access to the offical Raspberry Pi Camera module.
* ```Pi.Info``` Provides information on this Raspberry Pi's CPU and form factor.
* ```Pi.Gpio``` Provides access to the Raspberry Pi's GPIO as a collection of GPIO Pins.
* ```Pi.Spi``` Provides access to the 2-channel SPI bus.
* ```Pi.I2c``` Provides access to the functionality of the i2c bus.
* ```Pi.Timing``` Provides access to The PI's Timing and threading API.

_Please note you program needs to be run with ```sudo```. Example ```sudo mono myprogram.exe``` in order to work correctly._

This library depends on the wonderful ```WiringPi``` library avaialble [here](http://wiringpi.com/). You do not need to install this library yourself. The ```RaspberryIO``` assembly will automatically extract the compiled binary of the library in the same path as the entry assembly.

## NuGet Installation:
```
PM> Install-Package Unosquare.Raspberry.IO
```

## Running the latest version of Mono
It is recommended that you install the latest available release of Mono because what is available in the Raspbian repo is quite old (3.X). These commands were tested using Raspbian Jessie. The version of Mono that is installed at the time of this writing is:
``` 
Mono JIT compiler version 4.6.2 (Stable 4.6.2.7/08fd525 Mon Nov 14 12:43:54 UTC 2016) 
```

The commands to get Mono installed are the following:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install mono-complete
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
echo "deb http://download.mono-project.com/repo/debian wheezy main" | sudo tee /etc/apt/sources.list.d/mono-xamarin.list
sudo apt-get update
sudo apt-get dist-upgrade
```

Now, verify your version of Mono by running ```mono --version```. Version 4.6 and above should be good enough.

## The Camera Module
The ```Pi.Camera``` module uses ```raspivid``` and ```raspistill``` to access to camera so they must be installed in order for your program to work propely. ```raspistill``` arguments are specified in an instance of the ```CameraStillSettings``` class, while the ```raspivid``` arguments are specified in an instance of the ```CameraVideoSettings``` class. 

### Capturing Images
The ```Pi.Camera.CaptureImage*``` methods simply return an array of bytes containing the capture image. There are synchronous and asynchronous falvors of these methods so you can use the familiar ```async``` and ```await``` pattern to capture your images. All ```raspistill``` arguments (except for those that control user interaction such as ```-k```) are available via the ```CameraStillSettings```. To start, create a new instance of the ```CameraStillSettings``` class and pass it on to your choice of the ```Pi.Camera.CaptureImage*``` methods. There are shortcut methods available that simply take a JPEG image at the given Width and Height. By default, the shortcut methods set the JPEG quality at 90%.

Example using a shortcut method:
```csharp
static void TestCaptureImage()
{
    var pictureBytes = Pi.Camera.CaptureImageJpeg(640, 480);
    var targetPath = "/home/pi/picture.jpg";
    if (File.Exists(targetPath))
        File.Delete(targetPath);

    File.WriteAllBytes(targetPath, pictureBytes);
    Console.WriteLine($"Took picture -- Byte count: {pictureBytes.Length}");
}
```

Example using a CaptureImage method:
```csharp
// TODO: example code here
```

### Capturing Video
Capturing video streams is somewhat different but it is still very easy to do. The concept behind it is to _Open_ a video stream providing your own callback. When opening the stream ```Raspberry IO``` will spawn a separte thread and will not block the execution of your code, but it will continually call your callback method containing the bytes that are being read from the camera until the _Close_ method is called or until the timeout is reached.

Example of capturing a stream of H.264 video
```csharp
static void TestCaptureVideo()
{
    // Setup our working variables
    var videoByteCount = 0;
    var videoEventCount = 0;
    var startTime = DateTime.UtcNow;

    // Configure video settings
    var videoSettings = new CameraVideoSettings()
    {
        CaptureTimeoutMilliseconds = 0,
        CaptureDisplayPreview = false,
        ImageFlipVertically = true,
        CaptureExposure = CameraExposureMode.Night,
        CaptureWidth = 1920,
        CaptureHeight = 1080
    };

    try
    {
        // Start the video recording
        Pi.Camera.OpenVideoStream(videoSettings,
            onDataCallback: (data) => { videoByteCount += data.Length; videoEventCount++; },
            onExitCallback: null);

        // Wait for user interaction
        startTime = DateTime.UtcNow;
        Console.WriteLine("Press any key to stop reading the video stream . . .");
        Console.ReadKey(true);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"{ex.GetType()}: {ex.Message}");
    }
    finally
    {
        // Always close the video stream to ensure raspivid quits
        Pi.Camera.CloseVideoStream();

        // Output the stats
        var megaBytesReceived = (videoByteCount / (1024f * 1024f)).ToString("0.000");
        var recordedSeconds = DateTime.UtcNow.Subtract(startTime).TotalSeconds.ToString("0.000");
        Console.WriteLine($"Capture Stopped. Received {megaBytesReceived} Mbytes in {videoEventCount} callbacks in {recordedSeconds} seconds");
    }            
}
```

## Obtaining Board and System Information
```RaspberryIO``` contains useful utilities to obtain information about the board it is running on. You can simply call the ```Pi.Info.ToString()``` method to obtain a dump of all system properties as a single ```string```, or you can use the individual properties such as Installed RAM, Processor Count, Raspberry Pi Version, Serial Number, etc. There's not a lot more to this.
Please note ```Pi.Info``` depends on ```Wiring Pi```, and the ```/proc/cpuinfo``` and ```/proc/meminfo``` files.

## Using the GPIO Pins
TODO

### Pin Information
TODO

### Digital Read and Write
TODO

### Hardware PWM
TODO

### Software PWM
TODO

### Tone Generation
TODO

### Interrupts and Callbacks
TODO

## Using the SPI Bus
I really liked the following description from [Neil's Log Book](http://nrqm.ca/nrf24l01/serial-peripheral-interface/): _The SPI (Serial Peripheral Interface) protocol behaves like a ring buffer, so that whenever the master sends a byte to the slave, the slave sends a byte back to the master. The slave can use this behaviour to return a status byte, a response to a previous byte, or null data (the master may choose to read the returned byte, or ignore it). The bus operates on a 4-wire interface._

```RaspberryIO``` provides easy access to the 2 SPI channels available on the Raspberry. The functionality depends on ```Wiring Pi```'s SPI library. Please note that you may need to issue the command ```gpio load spi``` before starting your application (or as a ```System.Diagnostics.Process``` when your application starts) if the SPI kernel drivers have not been loaded.

In order to use an SPI channel you **MUST** always set the ```Channel0Frequency``` or ```Channel1Frequency``` (depending on the channel you want to use) before calling the ```SendReceive``` method. If the property is not set beforehand the SPI channel will fail initialization. See an example below: 

Example of using the SPI Bus
```csharp
    Pi.Spi.Channel0Frequency = SpiChannel.MinFrequency;
    var request = System.Text.Encoding.ASCII.GetBytes("HELLO!");
    var response = Pi.Spi.Channel0.SendReceive(request);
```

## I2C to connect ICs
The Inter IC Bus (I2C) is a cousin of the SPI bus but it is somewhat more complex and it does not work as a ring buffer like the SPI bus. It also connects all of its slave devices in series and depends on 2 lines only. There is a nice tutorial on setting up and using the I2C bus at [Robot Electronics](http://www.robot-electronics.co.uk/i2c-tutorial). From their site: _The physical bus is just two wires, called SCL and SDA. SCL is the clock line. It is used to synchronize all data transfers over the I2C bus. SDA is the data line. The SCL & SDA lines are connected to all devices on the I2C bus. There needs to be a third wire which is just the ground or 0 volts. There may also be a 5volt wire is power is being distributed to the devices. Both SCL and SDA lines are "open drain" drivers. What this means is that the chip can drive its output low, but it cannot drive it high. For the line to be able to go high you must provide pull-up resistors to the 5v supply. There should be a resistor from the SCL line to the 5v line and another from the SDA line to the 5v line. You only need one set of pull-up resistors for the whole I2C bus, not for each device._

```RaspberryIO``` provides easy access to the I2C bus available on the Raspberry. The functionality depends on ```Wiring Pi```'s I2C library. Please note that you may need to issue the command ```gpio load i2c``` before starting your application (or as a ```System.Diagnostics.Process``` when your application starts) if the I2C kernel drivers have not been loaded. The default baud rate is 100Kbps. If you wish to initialize the bus at a different baud rate you may issue for example, ```gpio load i2c 200```. This will load the bus at 200kbps.

In order to detect I2C devices you could use the ```i2cdetect``` system command. Just remember that on a Rev 1 Raspberry Pi it's device 0, and on a Rev. 2 it's device 1. e.g.
```
i2cdetect -y 0 # Rev 1
i2cdetect -y 1 # Rev 2
```

Example of using the I2C Bus
```csharp
// Register a device on the bus
var myDevice = Pi.I2c.AddDevice(0x20);

// Simple Write and Read (there are algo register read and write methods)
myDevice.Write(0x44);
var response = myDevice.Read();

// List registered devices on the I2C Bus
foreach (var device in Pi.I2c.Devices)
{
    Console.WriteLine($"Registered I2C Device: {device.DeviceId}");
}
```

## Timing and Threading
TODO

## Serial Ports (UART)
Where is the serial port API? Well, it is something we will most likely add in the future. For now, you can simply use the built-in ```SerialPort``` class the .NET framework provides.

## Similar Projects
- <a href="https://github.com/raspberry-sharp/raspberry-sharp-io">Raspberry# IO</a>
- <a href="https://github.com/danriches/WiringPi.Net">WiringPi.Net</a>
- <a href="https://github.com/andycb/PiSharp">PiSharp</a>