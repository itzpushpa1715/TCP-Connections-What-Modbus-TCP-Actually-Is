# TCP-Connections-What-Modbus-TCP-Actually-Is
First — What is TCP?
Think of TCP like a phone call:

You dial (connect to IP + Port)
The line opens (connection established)
You talk back and forth (send/receive data)
You hang up (disconnect)

In C# this is done with TcpClient:
csharpTcpClient client = new TcpClient();
client.Connect("127.0.0.1", 502);  // dial the PLC
```

Port **502** is the standard Modbus TCP port — like how websites use port 80.

---

### Second — What is Modbus TCP?

Modbus is just an **agreed language** between your app and the PLC.

Imagine you and the PLC agreed:
> "If you send me these specific bytes, I will send back the register values"

That agreement is the **Modbus protocol**.

Every message has two parts:
```
┌─────────────────────────────┬──────────────────────┐
│       MBAP Header (7 bytes) │    PDU (5 bytes)      │
│  Transaction │ Protocol │ Length │ Unit │ FC │ Data │
└─────────────────────────────┴──────────────────────┘

Third — The NetworkStream
After connecting, you get a stream — like a pipe between your app and PLC:
csharp// After connecting, get the stream
NetworkStream stream = client.GetStream();

// Send bytes through the pipe TO the PLC
stream.Write(myBytes, 0, myBytes.Length);

// Receive bytes FROM the PLC (async = don't freeze the UI)
stream.BeginRead(buffer, 0, buffer.Length, 
                 new AsyncCallback(ReceiveDataFromPLC), stream);
```

The `BeginRead` is **asynchronous** — it means:
> "Wait in the background for data, and when it arrives call `ReceiveDataFromPLC`"

This is why the UI doesn't freeze while waiting.

---

### Fourth — The Complete Picture

Here is exactly what happens when you click Read:
```
Your App                          PLC (Simulator)
   │                                    │
   │── sends 12 bytes (read request) ──►│
   │                                    │ processes request
   │◄── sends back response bytes ──────│
   │                                    │
   │  ReceiveDataFromPLC() fires        │
   │  you read _buffer[]               │
   │  update UI labels                 │

Fifth — Async and the UI Thread
This is very important and a common mistake:
csharpvoid ReceiveDataFromPLC(IAsyncResult ar)
{
    // ⚠️ This runs on a BACKGROUND thread
    // You CANNOT update UI directly here
    // This will CRASH:
    txtStatus.Text = "hello";  // ❌ CRASH!

    // You MUST use Dispatcher.Invoke to get back to UI thread:
    Dispatcher.Invoke(() =>
    {
        txtStatus.Text = "hello";  // ✅ WORKS!
    });
}

Now Let's Write Real Code
Open your MainWindow.xaml.cs. Let's complete the Connect button properly.
Replace your empty btnConnect_Click with this — type it yourself:
csharpprivate void btnConnect_Click(object sender, RoutedEventArgs e)
{
    try
    {
        // Read IP and Port from the UI textboxes
        string ip = txtIP.Text;
        int port = int.Parse(txtPort.Text);

        // Call the Connect method
        Connect(ip, port);

        // Update the UI to show connected
        ellipseStatus.Fill = System.Windows.Media.Brushes.Green;
        lblConnected.Content = "Connected!";
        btnConnect.IsEnabled = false; // prevent clicking again
    }
    catch (Exception ex)
    {
        MessageBox.Show("Could not connect: " + ex.Message);
        ellipseStatus.Fill = System.Windows.Media.Brushes.Red;
        lblConnected.Content = "Failed";
    }
}

Understanding the Connect Method
The Connect method in your template is already written. Let's understand it line by line:
csharpvoid Connect(string IP, int Port)
{
    // Only connect if not already connected
    if (!modbusTCPclient.Connected)
    {
        // Step 1: Open TCP connection to PLC
        modbusTCPclient.Connect(IP, Port);

        // Step 2: Get the communication pipe
        modbusStream = modbusTCPclient.GetStream();

        // Step 3: Start listening for incoming data
        // When data arrives → ReceiveDataFromPLC is called automatically
        modbusStream.BeginRead(_buffer, 0, _buffer.Length,
            new AsyncCallback(ReceiveDataFromPLC), modbusStream);
    }
    else
    {
        MessageBox.Show("Already connected!");
    }
}

Test This Now

Make sure your simulator is running (the .exe file from your uploads)
Build with Ctrl+Shift+B
Run with F5
Type 127.0.0.1 and 502
Click Connect
The circle should turn Green


✅ Quick Check Questions
Before Day 3, make sure you can answer:

What does TcpClient.Connect() do?
Why do we use Dispatcher.Invoke() inside ReceiveDataFromPLC?
What is BeginRead doing — why not just use regular Read?


Try running it and tell me:

Did it connect? (green light?)
Any errors?
Let's go! Day 3 is where it gets really satisfying — you'll actually see real data coming from the PLC simulator.

Day 3: Sending Modbus Bytes + Reading Register Values

First — Start the Simulator
Before writing any code, start your simulator:

Find PC-sov_simulaattori.exe in your files
Double click to run it
It listens on 127.0.0.1 port 502


Second — Understand the Full Read Request
Remember from earlier, a read request is 12 bytes:
Byte: [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] [11]

MBAP Header:
[0][1] = Transaction ID = 0x00, 0x01  (we choose this, any number)
[2][3] = Protocol ID   = 0x00, 0x00  (always 0 for Modbus)
[4][5] = Length        = 0x00, 0x06  (6 bytes follow after this)

PDU:
[6]    = Unit ID       = 0x01        (slave address)
[7]    = Function Code = 0x03        (Read Holding Registers)
[8][9] = Start Address = 0x00, 0x00  (start from HR0)
[10][11]= Quantity     = 0x00, 0x06  (read 6 registers)

Third — Understand the Response
The PLC sends back 21 bytes for 6 registers:
[0][1]  = Transaction ID (echo)
[2][3]  = Protocol ID (echo)
[4][5]  = Length
[6]     = Unit ID
[7]     = Function Code 0x03
[8]     = Byte Count (12 = 6 registers x 2 bytes)

Register values (2 bytes each, High byte first):
[9][10]  = HR0 Setpoint
[11][12] = HR1 Control Word
[13][14] = HR2 Status
[15][16] = HR3 Switches
[17][18] = HR4 Temperature
[19][20] = HR5 Cycle Counter

Fourth — Scaling Values
Your assignment table says values must be scaled:
Setpoint:    raw 0-1000  → display 0.0-100.0  → divide by 10
Temperature: raw 0-1000  → display 0.0-100.0  → divide by 10
Others:      no scaling needed

Now Write The Code
Step 1: Complete btnRead_Click
Type this into your MainWindow.xaml.cs:
csharpprivate void btnRead_Click(object sender, RoutedEventArgs e)
{
    try
    {
        // Check if connected first
        if (!modbusTCPclient.Connected)
        {
            MessageBox.Show("Not connected! Please connect first.");
            return;
        }

        // Build the 12-byte Modbus read request
        byte[] request = new byte[12];

        // MBAP Header
        request[0] = 0x00; // Transaction ID high
        request[1] = 0x01; // Transaction ID low
        request[2] = 0x00; // Protocol ID high
        request[3] = 0x00; // Protocol ID low
        request[4] = 0x00; // Length high
        request[5] = 0x06; // Length low

        // PDU
        request[6]  = 0x01; // Unit ID
        request[7]  = 0x03; // Function Code: Read Holding Registers
        request[8]  = 0x00; // Start address high (HR0 = address 0)
        request[9]  = 0x00; // Start address low
        request[10] = 0x00; // Number of registers high
        request[11] = 0x06; // Number of registers low (read 6)

        // Send to PLC
        RequestToPLC(request);
    }
    catch (Exception ex)
    {
        MessageBox.Show("Read failed: " + ex.Message);
    }
}

Step 2: Complete ReceiveDataFromPLC
This fires automatically when PLC responds. Type this:
csharpvoid ReceiveDataFromPLC(IAsyncResult ar)
{
    try
    {
        modbusStream.EndRead(ar);

        // Start listening again for next response
        modbusStream.BeginRead(_buffer, 0, _buffer.Length,
            new AsyncCallback(ReceiveDataFromPLC), modbusStream);

        // Must update UI on the UI thread
        Dispatcher.Invoke(() =>
        {
            // Show raw bytes for debugging
            txtRawResponse.Text = BitConverter.ToString(_buffer);

            // Only process if it is a valid read response (FC 03)
            if (_buffer[7] == 0x03)
            {
                // Extract raw register values
                // High byte shifted left 8 + low byte = full 16-bit value
                int setpoint     = (_buffer[9]  << 8) | _buffer[10];
                int controlWord  = (_buffer[11] << 8) | _buffer[12];
                int status       = (_buffer[13] << 8) | _buffer[14];
                int switches     = (_buffer[15] << 8) | _buffer[16];
                int temperature  = (_buffer[17] << 8) | _buffer[18];
                int cycleCounter = (_buffer[19] << 8) | _buffer[20];

                // Scale values into real world units
                double setpointReal = setpoint / 10.0;
                double tempReal     = temperature / 10.0;

                // Update text displays
                txtReadSetpoint.Text  = setpointReal.ToString("F1");
                txtControlWord.Text   = controlWord.ToString();
                txtStatus.Text        = status.ToString();
                txtSwitches.Text      = switches.ToString();
                txtTemperature.Text   = tempReal.ToString("F1");
                txtCycleCounter.Text  = cycleCounter.ToString();

                // Update temperature progress bar
                progressTemp.Value = tempReal;

                // Update status indicator color
                // Status: 0=stopped, 1=running, else=fault
                if (status == 0)
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Gray;
                else if (status == 1)
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Green;
                else
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Red;

                // Update switch indicators (8 bits)
                sw0.Fill = (switches & (1 << 0)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw1.Fill = (switches & (1 << 1)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw2.Fill = (switches & (1 << 2)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw3.Fill = (switches & (1 << 3)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw4.Fill = (switches & (1 << 4)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw5.Fill = (switches & (1 << 5)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw6.Fill = (switches & (1 << 6)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
                sw7.Fill = (switches & (1 << 7)) != 0 ? 
                    System.Windows.Media.Brushes.Green : 
                    System.Windows.Media.Brushes.Gray;
            }
        });
    }
    catch (Exception ex)
    {
        Dispatcher.Invoke(() =>
        {
            MessageBox.Show("Receive error: " + ex.Message);
        });
    }
}

Step 3: Also add txtControlWord to your XAML
You may have noticed txtControlWord — add this inside your Read GroupBox in XAML:
xml<Label Content="Control Word:"/>
<TextBox x:Name="txtControlWord" Width="80" 
         IsReadOnly="True" Margin="5,0"/>

Test It Now

Run the simulator
Build Ctrl+Shift+B → should say 0 errors
Press F5 to run
Click Connect → circle goes green
Click Read Registers
You should see values appearing in the text boxes!


✅ Understanding Check
Make sure you can explain these to your teacher:
Q: Why do we do (_buffer[9] << 8) | _buffer[10]?

Because one register is 16 bits = 2 bytes. << 8 shifts the high byte left by 8 positions, then | combines it with the low byte.

Q: Why Dispatcher.Invoke?

Because ReceiveDataFromPLC runs on a background thread. Only the UI thread can update UI elements.

Q: Why divide temperature by 10?

Because the PLC stores 25.5°C as integer 255. Dividing by 10 gives the real value.

That means Day 3 is working! Great progress. Let's go to Day 4.

Day 4: Writing Values TO the PLC

First — Understand Write Request
Writing is similar to reading but we use Function Code 06 (Write Single Register) and we send the value we want to write.
A write request is 12 bytes:
Byte: [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] [11]

MBAP Header:
[0][1] = Transaction ID = 0x00, 0x02  (different from read)
[2][3] = Protocol ID   = 0x00, 0x00  (always 0)
[4][5] = Length        = 0x00, 0x06  (6 bytes follow)

PDU:
[6]     = Unit ID       = 0x01        (slave address)
[7]     = Function Code = 0x06        (Write Single Register)
[8][9]  = Register Addr = 0x00, 0x00  (HR0 = Setpoint)
[10][11]= Value         = HIGH, LOW   (the value to write)

Second — Scaling for Writing
Remember the assignment table:
Setpoint: GUI shows 0.0-100.0 → PLC stores 0-1000
          So multiply by 10 before sending!

Control Word: 0 or 1 (no scaling needed)

Third — Splitting Value into Two Bytes
A register holds 16 bits = 2 bytes. You need to split your integer:
csharpint value = 500; // e.g. setpoint 50.0 × 10 = 500

// Split into high and low byte
byte high = (byte)(value >> 8);   // shifts right 8 = high byte
byte low  = (byte)(value & 0xFF); // keeps only low 8 bits

// Example: 500 = 0x01F4
// high = 0x01
// low  = 0xF4

Now Write The Code
Step 1: Complete btnWrite_Click
Type this into your MainWindow.xaml.cs:
csharpprivate void btnWrite_Click(object sender, RoutedEventArgs e)
{
    try
    {
        // Check connection first
        if (!modbusTCPclient.Connected)
        {
            MessageBox.Show("Not connected! Please connect first.");
            return;
        }

        // Read setpoint from UI and validate
        if (!double.TryParse(txtSetpoint.Text, out double setpointGUI))
        {
            MessageBox.Show("Invalid setpoint value!");
            return;
        }

        // Validate range
        if (setpointGUI < 0 || setpointGUI > 100)
        {
            MessageBox.Show("Setpoint must be between 0 and 100!");
            return;
        }

        // Scale: GUI 0-100 → PLC 0-1000
        int setpointScaled = (int)(setpointGUI * 10);

        // Split into two bytes
        byte setpointHigh = (byte)(setpointScaled >> 8);
        byte setpointLow  = (byte)(setpointScaled & 0xFF);

        // Build write request for HR0 (Setpoint)
        byte[] request = new byte[12];

        // MBAP Header
        request[0] = 0x00; // Transaction ID high
        request[1] = 0x02; // Transaction ID low (different from read)
        request[2] = 0x00; // Protocol ID high
        request[3] = 0x00; // Protocol ID low
        request[4] = 0x00; // Length high
        request[5] = 0x06; // Length low

        // PDU
        request[6]  = 0x01;         // Unit ID
        request[7]  = 0x06;         // Function Code: Write Single Register
        request[8]  = 0x00;         // Register address high (HR0)
        request[9]  = 0x00;         // Register address low
        request[10] = setpointHigh; // Value high byte
        request[11] = setpointLow;  // Value low byte

        // Send to PLC
        RequestToPLC(request);

        MessageBox.Show("Setpoint written successfully!");
    }
    catch (Exception ex)
    {
        MessageBox.Show("Write failed: " + ex.Message);
    }
}

Step 2: Write Control Word to HR1
Now add writing the control word. Create a separate helper method for writing any register — this is cleaner and reusable:
csharpprivate void WriteRegister(int registerAddress, int value)
{
    try
    {
        // Check connection
        if (!modbusTCPclient.Connected)
        {
            MessageBox.Show("Not connected!");
            return;
        }

        // Split address into bytes
        byte addrHigh = (byte)(registerAddress >> 8);
        byte addrLow  = (byte)(registerAddress & 0xFF);

        // Split value into bytes
        byte valHigh = (byte)(value >> 8);
        byte valLow  = (byte)(value & 0xFF);

        // Build request
        byte[] request = new byte[12];
        request[0]  = 0x00;      // Transaction ID high
        request[1]  = 0x02;      // Transaction ID low
        request[2]  = 0x00;      // Protocol ID high
        request[3]  = 0x00;      // Protocol ID low
        request[4]  = 0x00;      // Length high
        request[5]  = 0x06;      // Length low
        request[6]  = 0x01;      // Unit ID
        request[7]  = 0x06;      // Function Code: Write Single Register
        request[8]  = addrHigh;  // Register address high
        request[9]  = addrLow;   // Register address low
        request[10] = valHigh;   // Value high
        request[11] = valLow;    // Value low

        RequestToPLC(request);
    }
    catch (Exception ex)
    {
        MessageBox.Show("Write error: " + ex.Message);
    }
}
Now update btnWrite_Click to also write control word using this helper:
csharp// After writing setpoint, also write control word to HR1
if (!int.TryParse(txtControlWord.Text, out int controlWord))
{
    MessageBox.Show("Invalid control word!");
    return;
}

if (controlWord != 0 && controlWord != 1)
{
    MessageBox.Show("Control word must be 0 or 1!");
    return;
}

// HR1 = address 1, no scaling needed
WriteRegister(1, controlWord);

Step 3: Update XAML for Control Word input
Make sure your Write GroupBox in XAML has an input for control word. It should already be there from Day 1 setup. If not add:
xml<Label Content="Control Word (0 or 1):"/>
<TextBox x:Name="txtControlWord" Width="60" 
         Text="0" Margin="5,0"/>

Step 4: Update ReceiveDataFromPLC for Write Response
When you write, the PLC echoes back the same bytes. Update your if statement in ReceiveDataFromPLC to handle both:
csharp// Handle read response
if (_buffer[7] == 0x03)
{
    // your existing read parsing code here
}
// Handle write response
else if (_buffer[7] == 0x06)
{
    // Write was successful - just update raw response display
    txtRawResponse.Text = "Write OK: " + BitConverter.ToString(_buffer);
}

Test It Now

Run simulator
Build Ctrl+Shift+B
Press F5
Connect
Type 50.5 in setpoint box
Type 1 in control word box
Click Write
Click Read → setpoint should now show 50.5


✅ What You Can Now Explain to Teacher
Q: What is Function Code 06?

Write Single Register — writes one 16-bit value to a specific register address

Q: Why do you multiply setpoint by 10?

The PLC stores values as integers. 50.5 becomes 505 in the PLC. When reading back we divide by 10.

Q: What is value >> 8?

It shifts the bits right by 8 positions to get the high byte of a 16-bit number

Q: What is value & 0xFF?

It masks the lower 8 bits to get just the low byte

Let's go! Day 5 is where your app becomes truly automatic — values update on their own every second without clicking Read.

Day 5: DispatcherTimer — Automatic Cyclic Reading

First — What is DispatcherTimer?
Think of it like an alarm clock that fires every second:
Time 0s  → Read registers
Time 1s  → Read registers  
Time 2s  → Read registers
Time 3s  → Read registers
... forever until you stop it
The assignment specifically says to use DispatcherTimer because it runs on the UI thread — meaning you can update UI elements directly without Dispatcher.Invoke!

Second — How DispatcherTimer Works
csharp// 1. Create the timer
DispatcherTimer timer = new DispatcherTimer();

// 2. Set how often it fires
timer.Interval = TimeSpan.FromSeconds(1);

// 3. Tell it what to do when it fires
timer.Tick += Timer_Tick;

// 4. Start it
timer.Start();

// 5. Stop it
timer.Stop();

Now Write The Code
Step 1: Declare Timer at the Top
Add this with your other field declarations at the top of your class:
csharp// Add this with your other private fields at top of class
private DispatcherTimer _readTimer = new DispatcherTimer();
private bool _isLogging = false;

Step 2: Set Up Timer in Constructor
Update your MainWindow() constructor:
csharppublic MainWindow()
{
    InitializeComponent();

    // Set timer interval to 1 second
    _readTimer.Interval = TimeSpan.FromSeconds(1);

    // Tell timer what method to call every second
    _readTimer.Tick += ReadTimer_Tick;
}

Step 3: Create the Timer Tick Method
This is called every second. Type this:
csharpprivate void ReadTimer_Tick(object sender, EventArgs e)
{
    // Only read if connected
    if (modbusTCPclient.Connected)
    {
        // Build read request - same as btnRead_Click
        byte[] request = new byte[12];

        request[0]  = 0x00;
        request[1]  = 0x01;
        request[2]  = 0x00;
        request[3]  = 0x00;
        request[4]  = 0x00;
        request[5]  = 0x06;
        request[6]  = 0x01;
        request[7]  = 0x03; // Read Holding Registers
        request[8]  = 0x00;
        request[9]  = 0x00;
        request[10] = 0x00;
        request[11] = 0x06;

        RequestToPLC(request);
    }
}

Step 4: Start Timer After Connecting
Update your btnConnect_Click to start the timer after connecting:
csharpprivate void btnConnect_Click(object sender, RoutedEventArgs e)
{
    try
    {
        string ip = txtIP.Text;
        int port = int.Parse(txtPort.Text);

        Connect(ip, port);

        // Update UI
        ellipseStatus.Fill = System.Windows.Media.Brushes.Green;
        lblConnected.Content = "Connected!";
        btnConnect.IsEnabled = false;

        // Start cyclic reading automatically after connect
        _readTimer.Start();
    }
    catch (Exception ex)
    {
        MessageBox.Show("Could not connect: " + ex.Message);
        ellipseStatus.Fill = System.Windows.Media.Brushes.Red;
        lblConnected.Content = "Failed";
    }
}

Step 5: Clean Up ReceiveDataFromPLC
Now that timer handles reading automatically you can simplify. Since DispatcherTimer runs on UI thread you still keep Dispatcher.Invoke in ReceiveDataFromPLC because that method is still triggered by async network read which runs on background thread:
csharpvoid ReceiveDataFromPLC(IAsyncResult ar)
{
    try
    {
        modbusStream.EndRead(ar);

        // Listen again for next response
        modbusStream.BeginRead(_buffer, 0, _buffer.Length,
            new AsyncCallback(ReceiveDataFromPLC), modbusStream);

        Dispatcher.Invoke(() =>
        {
            // Show raw bytes
            txtRawResponse.Text = BitConverter.ToString(_buffer);

            // Read response
            if (_buffer[7] == 0x03)
            {
                // Parse values
                int setpoint     = (_buffer[9]  << 8) | _buffer[10];
                int controlWord  = (_buffer[11] << 8) | _buffer[12];
                int status       = (_buffer[13] << 8) | _buffer[14];
                int switches     = (_buffer[15] << 8) | _buffer[16];
                int temperature  = (_buffer[17] << 8) | _buffer[18];
                int cycleCounter = (_buffer[19] << 8) | _buffer[20];

                // Scale
                double setpointReal = setpoint / 10.0;
                double tempReal     = temperature / 10.0;

                // Update displays
                txtReadSetpoint.Text  = setpointReal.ToString("F1");
                txtStatus.Text        = status.ToString();
                txtSwitches.Text      = switches.ToString();
                txtTemperature.Text   = tempReal.ToString("F1");
                txtCycleCounter.Text  = cycleCounter.ToString();
                progressTemp.Value    = tempReal;

                // Status light
                if (status == 0)
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Gray;
                else if (status == 1)
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Green;
                else
                    ellipseStatus.Fill = 
                        System.Windows.Media.Brushes.Red;

                // Switch lights
                sw0.Fill = (switches & (1 << 0)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw1.Fill = (switches & (1 << 1)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw2.Fill = (switches & (1 << 2)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw3.Fill = (switches & (1 << 3)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw4.Fill = (switches & (1 << 4)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw5.Fill = (switches & (1 << 5)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw6.Fill = (switches & (1 << 6)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;
                sw7.Fill = (switches & (1 << 7)) != 0 ?
                    System.Windows.Media.Brushes.Green :
                    System.Windows.Media.Brushes.Gray;

                // Log to CSV if logging is active
                if (_isLogging)
                {
                    LogData(setpointReal, controlWord, status,
                            switches, tempReal, cycleCounter);
                }
            }
            else if (_buffer[7] == 0x06)
            {
                txtRawResponse.Text = 
                    "Write OK: " + BitConverter.ToString(_buffer);
            }
        });
    }
    catch (Exception ex)
    {
        Dispatcher.Invoke(() =>
        {
            txtRawResponse.Text = "Error: " + ex.Message;
        });
    }
}

Test It Now

Start simulator
Build Ctrl+Shift+B
Press F5
Click Connect
Watch the values — they should update automatically every second without clicking Read!
Try writing a new setpoint and watch it update automatically


✅ What You Can Explain to Teacher
Q: Why DispatcherTimer instead of a regular Timer?

DispatcherTimer runs on the UI thread so it is safe to update UI elements directly. A regular Timer runs on a background thread and would require Dispatcher.Invoke.

Q: Why 1 second interval?

The assignment says not to make it too fast. Too frequent reads increase communication load and can cause errors if new commands arrive before previous ones finish.

Q: What happens if connection drops during timer?

The modbusTCPclient.Connected check prevents sending if disconnected. The try/catch handles any unexpected errors.


Let's go! Day 6 is the last major feature — saving data to a CSV file.

Day 6: CSV Data Logging

First — What is a CSV File?
CSV = Comma Separated Values. It's a plain text file that Excel can open:
Timestamp;Setpoint;ControlWord;Status;Switches;Temperature;CycleCounter
2026-03-29 12:00:01;50.0;1;1;5;25.5;100
2026-03-29 12:00:02;50.0;1;1;5;25.6;101
2026-03-29 12:00:03;50.0;1;1;5;25.7;102
The assignment says use semicolon as delimiter.

Second — What We Need
From the assignment requirements:

Button to start logging
Button to stop logging
Text field showing save path
Path selectable via dialog
CSV initialized with column headers on first start
Logging runs every second when active


Now Write The Code
Step 1: Add File Path Variable
Add this with your other fields at the top of your class:
csharpprivate string _logFilePath = "";

Step 2: Browse Button
This opens a save file dialog so user can choose where to save:
csharpprivate void btnBrowse_Click(object sender, RoutedEventArgs e)
{
    // Open save file dialog
    Microsoft.Win32.SaveFileDialog dialog = 
        new Microsoft.Win32.SaveFileDialog();

    // Set file filter to CSV only
    dialog.Filter = "CSV Files (*.csv)|*.csv";
    dialog.FileName = "ModbusLog";
    dialog.DefaultExt = ".csv";

    // Show dialog and check if user clicked Save
    bool? result = dialog.ShowDialog();

    if (result == true)
    {
        // Save the chosen path
        _logFilePath = dialog.FileName;

        // Show path in text box
        txtFilePath.Text = _logFilePath;
    }
}

Step 3: Start Logging Button
csharpprivate void btnStartLog_Click(object sender, RoutedEventArgs e)
{
    // Check if path is selected
    if (string.IsNullOrEmpty(_logFilePath))
    {
        MessageBox.Show("Please select a file path first!");
        return;
    }

    // Check if connected
    if (!modbusTCPclient.Connected)
    {
        MessageBox.Show("Please connect to PLC first!");
        return;
    }

    try
    {
        // Write header line to CSV file
        // This creates the file if it doesn't exist
        using (System.IO.StreamWriter writer = 
               new System.IO.StreamWriter(_logFilePath, false))
        {
            writer.WriteLine(
                "Timestamp;Setpoint;ControlWord;Status;" +
                "Switches;Temperature;CycleCounter");
        }

        // Start logging flag
        _isLogging = true;

        // Update buttons
        btnStartLog.IsEnabled = false;
        btnStopLog.IsEnabled  = true;
        btnBrowse.IsEnabled   = false;

        MessageBox.Show("Logging started!");
    }
    catch (Exception ex)
    {
        MessageBox.Show("Could not start logging: " + ex.Message);
    }
}

Step 4: Stop Logging Button
csharpprivate void btnStopLog_Click(object sender, RoutedEventArgs e)
{
    // Stop logging
    _isLogging = false;

    // Update buttons
    btnStartLog.IsEnabled = true;
    btnStopLog.IsEnabled  = false;
    btnBrowse.IsEnabled   = true;

    MessageBox.Show("Logging stopped! File saved at:\n" + _logFilePath);
}

Step 5: LogData Method
This is called every second from ReceiveDataFromPLC when logging is active:
csharpprivate void LogData(double setpoint, int controlWord, int status,
                     int switches, double temperature, int cycleCounter)
{
    try
    {
        // Get current timestamp
        string timestamp = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");

        // Build one line of CSV data
        string line = string.Format("{0};{1};{2};{3};{4};{5};{6}",
            timestamp,
            setpoint.ToString("F1"),
            controlWord,
            status,
            switches,
            temperature.ToString("F1"),
            cycleCounter);

        // Append line to file
        // true = append, don't overwrite
        using (System.IO.StreamWriter writer =
               new System.IO.StreamWriter(_logFilePath, true))
        {
            writer.WriteLine(line);
        }
    }
    catch (Exception ex)
    {
        // Don't crash the app if logging fails
        // Just show in raw response box
        txtRawResponse.Text = "Log error: " + ex.Message;
        _isLogging = false;
        btnStartLog.IsEnabled = true;
        btnStopLog.IsEnabled  = false;
    }
}

Step 6: Make Sure LogData is Called
In your ReceiveDataFromPLC inside Dispatcher.Invoke, you already have this from Day 5:
csharp// Log to CSV if logging is active
if (_isLogging)
{
    LogData(setpointReal, controlWord, status,
            switches, tempReal, cycleCounter);
}
Make sure this line is there inside your if (_buffer[7] == 0x03) block.

Step 7: Update XAML Buttons
Make sure your logging GroupBox in XAML has the correct button names. It should already match from Day 1. Double check:
xml<GroupBox Header="Data Logging" Grid.Row="3" Margin="0,0,0,10">
    <StackPanel Orientation="Horizontal" Margin="5">
        <TextBox x:Name="txtFilePath" Width="300" 
                 IsReadOnly="True" Text="No file selected" 
                 Margin="0,0,5,0"/>
        <Button x:Name="btnBrowse" Content="Browse" 
                Width="70" Click="btnBrowse_Click" 
                Margin="0,0,5,0"/>
        <Button x:Name="btnStartLog" Content="Start Logging"
                Width="100" Click="btnStartLog_Click" 
                Margin="0,0,5,0"/>
        <Button x:Name="btnStopLog" Content="Stop Logging"
                Width="100" Click="btnStopLog_Click" 
                IsEnabled="False"/>
    </StackPanel>
</GroupBox>
```

---

## Test It Now

1. Start simulator
2. Build **Ctrl+Shift+B** → 0 errors
3. Press **F5**
4. Click **Connect**
5. Click **Browse** → choose a location → name it `test.csv` → Save
6. Click **Start Logging**
7. Wait 5-10 seconds
8. Click **Stop Logging**
9. Open the CSV file in Excel or Notepad
10. You should see rows of data with timestamps!

---

## ✅ What You Can Explain to Teacher

**Q: What is a CSV file?**
> A plain text file where values are separated by a delimiter — semicolon in this case. Excel can open it as a table.

**Q: Why `append: true` in StreamWriter?**
> So each new data row is added to the end of the file instead of overwriting everything.

**Q: Why write headers only when starting logging?**
> Headers should appear only once at the top of the file. Writing them every second would duplicate them on every row.

**Q: What happens if logging fails?**
> The catch block stops logging gracefully and shows an error instead of crashing the app.

---

## What's Left

You now have a **complete working application**! 🎉
```
✅ Connect to PLC
✅ Read all registers
✅ Write setpoint and control word  
✅ Cyclic auto-update every second
✅ Visual indicators (lights, progress bar)
✅ CSV data logging




Let's go! Day 7 is about making your app look professional and preparing you to confidently present it to your teacher.

Day 7: UI Polish + Presentation Preparation

Part 1: Polish the UI
Step 1: Update Your XAML for Better Look
Replace your entire MainWindow.xaml with this improved version:
```
xml<Window x:Class="Laaja_harkka_wpf_cs.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="PLC Monitor - Modbus TCP Client" 
        Height="700" Width="900"
        Background="#F0F0F0">

    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- HEADER -->
        <TextBlock Grid.Row="0" 
                   Text="PLC Monitor — Modbus TCP Client"
                   FontSize="20" FontWeight="Bold"
                   Margin="0,0,0,15"
                   Foreground="#2C3E50"/>

        <!-- CONNECTION SECTION -->
        <GroupBox Header="Connection" Grid.Row="1" 
                  Margin="0,0,0,10"
                  BorderBrush="#2C3E50">
            <StackPanel Orientation="Horizontal" Margin="8">
                <Label Content="IP Address:" 
                       VerticalAlignment="Center"/>
                <TextBox x:Name="txtIP" Width="130" 
                         Text="127.0.0.1" Margin="5,0"
                         VerticalContentAlignment="Center"
                         Height="28"/>
                <Label Content="Port:" 
                       VerticalAlignment="Center"/>
                <TextBox x:Name="txtPort" Width="60" 
                         Text="502" Margin="5,0"
                         VerticalContentAlignment="Center"
                         Height="28"/>
                <Button x:Name="btnConnect" 
                        Content="Connect" Width="90"
                        Height="28"
                        Click="btnConnect_Click" 
                        Margin="10,0"
                        Background="#27AE60"
                        Foreground="White"
                        FontWeight="Bold"/>
                <Ellipse x:Name="ellipseStatus" 
                         Width="22" Height="22"
                         Fill="Red" Margin="10,0,5,0"/>
                <Label x:Name="lblConnected" 
                       Content="Disconnected"
                       VerticalAlignment="Center"
                       FontWeight="Bold"
                       Foreground="#E74C3C"/>
            </StackPanel>
        </GroupBox>

        <!-- WRITE SECTION -->
        <GroupBox Header="Write to PLC" Grid.Row="2" 
                  Margin="0,0,0,10"
                  BorderBrush="#2C3E50">
            <StackPanel Orientation="Horizontal" Margin="8">
                <Label Content="Setpoint (0.0 - 100.0):" 
                       VerticalAlignment="Center"/>
                <TextBox x:Name="txtSetpoint" Width="90" 
                         Text="0.0" Margin="5,0"
                         VerticalContentAlignment="Center"
                         Height="28"/>
                <Label Content="Control Word (0/1):" 
                       VerticalAlignment="Center"/>
                <TextBox x:Name="txtControlWord" Width="60" 
                         Text="0" Margin="5,0"
                         VerticalContentAlignment="Center"
                         Height="28"/>
                <Button x:Name="btnWrite" 
                        Content="Write to PLC" Width="110"
                        Height="28"
                        Click="btnWrite_Click" 
                        Margin="10,0"
                        Background="#2980B9"
                        Foreground="White"
                        FontWeight="Bold"/>
            </StackPanel>
        </GroupBox>

        <!-- READ SECTION -->
        <GroupBox Header="PLC Register Values" Grid.Row="3" 
                  Margin="0,0,0,10"
                  BorderBrush="#2C3E50">
            <Grid Margin="8">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>

                <!-- Left column: numeric values -->
                <StackPanel Grid.Column="0">

                    <!-- Setpoint -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,5">
                        <Label Content="Setpoint:" Width="120"/>
                        <TextBox x:Name="txtReadSetpoint" 
                                 Width="90" IsReadOnly="True"
                                 Background="#ECF0F1"
                                 Height="26"
                                 VerticalContentAlignment="Center"/>
                    </StackPanel>

                    <!-- Status -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,5">
                        <Label Content="Status:" Width="120"/>
                        <TextBox x:Name="txtStatus" 
                                 Width="90" IsReadOnly="True"
                                 Background="#ECF0F1"
                                 Height="26"
                                 VerticalContentAlignment="Center"/>
                        <Ellipse x:Name="ellipseDeviceStatus"
                                 Width="22" Height="22"
                                 Fill="Gray" Margin="10,0"/>
                    </StackPanel>

                    <!-- Switches raw value -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,5">
                        <Label Content="Switches (raw):" Width="120"/>
                        <TextBox x:Name="txtSwitches" 
                                 Width="90" IsReadOnly="True"
                                 Background="#ECF0F1"
                                 Height="26"
                                 VerticalContentAlignment="Center"/>
                    </StackPanel>

                    <!-- Temperature -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,5">
                        <Label Content="Temperature (°C):" Width="120"/>
                        <TextBox x:Name="txtTemperature" 
                                 Width="90" IsReadOnly="True"
                                 Background="#ECF0F1"
                                 Height="26"
                                 VerticalContentAlignment="Center"/>
                    </StackPanel>

                    <!-- Cycle Counter -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,5">
                        <Label Content="Cycle Counter:" Width="120"/>
                        <TextBox x:Name="txtCycleCounter" 
                                 Width="90" IsReadOnly="True"
                                 Background="#ECF0F1"
                                 Height="26"
                                 VerticalContentAlignment="Center"/>
                    </StackPanel>

                    <!-- Temperature Bar -->
                    <StackPanel Orientation="Horizontal" 
                                Margin="0,10">
                        <Label Content="Temp:" Width="120"/>
                        <ProgressBar x:Name="progressTemp" 
                                     Width="200" Height="22"
                                     Minimum="0" Maximum="100"
                                     Foreground="#E74C3C"/>
                    </StackPanel>

                </StackPanel>

                <!-- Right column: switch indicators -->
                <StackPanel Grid.Column="1" Margin="20,0">
                    <Label Content="Switch States:" 
                           FontWeight="Bold" Margin="0,0,0,8"/>

                    <!-- Switch indicators in a grid -->
                    <UniformGrid Columns="4" Width="200">
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw0" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW0" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw1" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW1" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw2" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW2" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw3" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW3" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw4" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW4" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw5" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW5" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw6" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW6" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                        <StackPanel Margin="5">
                            <Ellipse x:Name="sw7" Width="30" 
                                     Height="30" Fill="Gray"/>
                            <TextBlock Text="SW7" 
                                       HorizontalAlignment="Center"
                                       FontSize="11"/>
                        </StackPanel>
                    </UniformGrid>
                </StackPanel>
            </Grid>
        </GroupBox>

        <!-- BOTTOM SECTION: Logging + Debug -->
        <Grid Grid.Row="4">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="*"/>
            </Grid.RowDefinitions>

            <!-- LOGGING -->
            <GroupBox Header="Data Logging" Grid.Row="0"
                      Margin="0,0,0,10"
                      BorderBrush="#2C3E50">
                <StackPanel Orientation="Horizontal" Margin="8">
                    <TextBox x:Name="txtFilePath" Width="280" 
                             IsReadOnly="True"
                             Text="No file selected" 
                             Margin="0,0,5,0"
                             VerticalContentAlignment="Center"
                             Height="28"
                             Background="#ECF0F1"/>
                    <Button x:Name="btnBrowse" 
                            Content="Browse..." Width="80"
                            Height="28"
                            Click="btnBrowse_Click" 
                            Margin="0,0,5,0"/>
                    <Button x:Name="btnStartLog" 
                            Content="Start Logging"
                            Width="110" Height="28"
                            Click="btnStartLog_Click" 
                            Margin="0,0,5,0"
                            Background="#27AE60"
                            Foreground="White"
                            FontWeight="Bold"/>
                    <Button x:Name="btnStopLog" 
                            Content="Stop Logging"
                            Width="110" Height="28"
                            Click="btnStopLog_Click" 
                            IsEnabled="False"
                            Background="#E74C3C"
                            Foreground="White"
                            FontWeight="Bold"/>
                </StackPanel>
            </GroupBox>

            <!-- DEBUG -->
            <GroupBox Header="Raw Response (debug)" 
                      Grid.Row="1"
                      BorderBrush="#2C3E50">
                <TextBox x:Name="txtRawResponse" 
                         IsReadOnly="True"
                         TextWrapping="Wrap" 
                         VerticalScrollBarVisibility="Auto"
                         Background="#ECF0F1"
                         FontFamily="Consolas"
                         FontSize="11"/>
            </GroupBox>
        </Grid>
    </Grid>
</Window>
```

Step 2: Update lblConnected in btnConnect_Click
Make the connected label green when connected:
csharplblConnected.Content = "Connected!";
lblConnected.Foreground = System.Windows.Media.Brushes.Green;

Step 3: Add Device Status Ellipse to ReceiveDataFromPLC
You now have ellipseDeviceStatus for the device status separately from the connection status. Update your status section:
csharp// Device status indicator (separate from connection indicator)

```
if (status == 0)
{
    ellipseDeviceStatus.Fill = 
        System.Windows.Media.Brushes.Gray;    // stopped
    txtStatus.Text = "0 - Stopped";
}
else if (status == 1)
{
    ellipseDeviceStatus.Fill = 
        System.Windows.Media.Brushes.Green;   // running
    txtStatus.Text = "1 - Running";
}
else
{
    ellipseDeviceStatus.Fill = 
        System.Windows.Media.Brushes.Red;     // fault
    txtStatus.Text = status + " - Fault";
}
```



## Part 2: Presentation Preparation

Here are the **exact questions your teacher will likely ask** and the answers you should know:



### Q1: What is Modbus TCP?
> Modbus TCP is a communication protocol used in industrial automation. It runs over TCP/IP on port 502. My application acts as the **client/master** and the PLC acts as the **server/slave**. The client sends requests and the server responds.



### Q2: Walk me through what happens when you click Connect
> The app reads the IP and port from the text boxes, creates a TCP connection using `TcpClient.Connect()`, gets the network stream, and starts asynchronous reading with `BeginRead` to listen for incoming data without blocking the UI.



### Q3: What is the structure of a Modbus TCP message?
> It has two parts — the MBAP header and the PDU. The header is 7 bytes containing transaction ID, protocol ID, length and unit ID. The PDU contains the function code and data. For reading registers I use function code 03, for writing I use function code 06.



### Q4: How do you read register values from the response?
> The response bytes come into `_buffer`. Each register is 2 bytes. I combine them using bit shifting — high byte shifted left by 8, OR'd with the low byte. For example `(_buffer[9] << 8) | _buffer[10]` gives me the first register value.



### Q5: Why do you scale the values?
> Modbus registers hold 16-bit integers only — no decimals. So the PLC stores temperature 25.5°C as integer 255. I divide by 10 when reading to get the real value. When writing a setpoint I multiply by 10 to convert back.



### Q6: What is DispatcherTimer and why use it?
> DispatcherTimer is a WPF timer that runs on the UI thread. I use it to send a read request every second automatically. Because it runs on the UI thread, I can update UI elements directly in its Tick event without threading issues.



### Q7: How does the switch register work?
> HR3 holds 8 switch states packed into one 16-bit integer. Each bit represents one switch. I use bitwise AND with a shifted 1 to check each bit — for example `(switches & (1 << 3)) != 0` checks if switch 3 is ON.



### Q8: How does CSV logging work?
> When logging starts, I create the file and write the column headers. Then every second when new data arrives, I append a new line with timestamp and all register values separated by semicolons. StreamWriter with append:true adds to the file without overwriting.



### Q9: How do you handle errors?
> I wrap all network operations in try/catch blocks. Connection errors show a message box. Receive errors are shown in the debug text box. Input validation prevents invalid values from being sent to the PLC.



### Q10: What is BeginRead and why use it?
> BeginRead is asynchronous — it waits for incoming data in the background without freezing the UI. When data arrives it automatically calls `ReceiveDataFromPLC`. I use `Dispatcher.Invoke` inside that method to safely update UI from the background thread.



## Part 3: Final Checklist

Go through this before your presentation:
```
✅ App connects to simulator with green indicator
✅ Values update automatically every second
✅ Setpoint write works and reads back correctly
✅ Control word 0 stops, 1 starts (status changes)
✅ Temperature progress bar moves with value
✅ Switch lights turn green/gray correctly
✅ Browse opens file dialog
✅ Start logging creates CSV with headers
✅ Stop logging saves file correctly
✅ CSV opens correctly in Excel
✅ App doesn't crash on invalid inputs
✅ Raw response box shows byte values
```



## 🎉 You Are Done!

Your complete application has:
```
Day 1 ✅ C# fundamentals
Day 2 ✅ TCP + Modbus understanding  
Day 3 ✅ Reading register values
Day 4 ✅ Writing register values
Day 5 ✅ Cyclic auto-update
Day 6 ✅ CSV data logging
Day 7 ✅ Polished UI + presentation ready
Build one final time with Ctrl+Shift+B, run it, test everything, and you are ready for your lab presentation!