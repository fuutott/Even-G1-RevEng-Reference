G1 AR Glasses Reverse Engineering Reference Document
Last Updated: December 30, 2024

Table of Contents
Introduction
Firmware Information
Firmware Versions
Firmware Extraction & Analysis
Bluetooth Low Energy (BLE) Communication
ANCS (Apple Notification Center Service)
G1 Command Structure
Packet Structures
Commands Overview
General Commands
Dashboard Commands (0x06)
Teleprompter (0x09) & Navigation (0x0A)
Translation Commands (0x0D, 0x0E, 0x0F)
Silent Mode (0x03)
Other Notable Commands
Notification Whitelisting
App Interaction & Whitelisting
Image Handling
Bitmap (BMP) Handling
Navigation Map Rendering
Logging & Debugging
Enabling Debug Mode
Log Prefixes and Sources
Synchronization & Communication Issues
Master-Slave Behavior
Desynchronization Causes
Command Redundancy
Development Tools & Resources
Firmware Analysis Tools
Libraries & Repositories
Community Resources
Miscellaneous Notes
Future Work & Considerations
Introduction
This document compiles the collective findings and observations from various contributors engaged in reverse engineering the G1 AR glasses. It covers firmware details, BLE communication protocols, command structures, image handling, synchronization mechanisms, and more. The aim is to provide a consolidated reference for ongoing and future reverse engineering efforts.

Firmware Information
Firmware Versions
1.3.7: An earlier firmware version referenced during initial reverse engineering attempts.
1.4.1: Identified as the current firmware version at the time of the latest updates.
1.4.5: A newer firmware version available for download.
Firmware Extraction & Analysis
Extraction Methods:
APK Decompilation: Initial firmware extraction involved decompiling the APK to access embedded firmware.
Filesystem Access: Accessed via rooted devices, particularly on Android (e.g., Pixel phones). Firmware files located at /data/data/com.even.g1/files.
Analysis Tools:
IDA Pro: Used for disassembling the firmware binary.
Ghidra: Suggested for firmware decompiling.
Ghidra with nRF5340 DK: For deeper analysis with specific hardware headers.
Observations:
The firmware contains multiple debug strings aiding in navigation and other functionalities.
Native Flutter applications add complexity to reverse engineering due to additional layers beyond Java bytecode.
RLE (Run-Length Encoding) compression is utilized specifically in navigation features.
Bluetooth Low Energy (BLE) Communication
ANCS (Apple Notification Center Service)
Purpose: Enables GATT BLE devices to receive notifications from iOS devices.
Implementation:
Handled entirely within the firmware (FW).
Questions remain about enabling ANCS mode when using custom iOS applications.
Reference: Apple ANCS Specification
G1 Command Structure
General Format:
css
Copy code
[Command ID][Packet Length][Sequence ID][Subcommand][Data...]
Example:
css
Copy code
0x06 0x15 0x00 [seq_id] 0x01 [timestamp][weather data]...
Packet Structures
Dashboard Update Packet (Command 0x06)
Structure:

bash
Copy code
[
    0x06,                # Command ID: Dashboard
    0x15, 0x00,          # Packet Length: 0x0015 (21 bytes)
    seq_id,              # Sequence Number
    0x01,                # Subcommand: Update Time and Weather
    0x4F, 0x6F, 0x71, 0x67,  # 32-bit Timestamp (e.g., 0x67716F4F)
    0x98, 0xC9, 0xAC, 0x31, 0x41, 0x19, 0x00, 0x00,  # 64-bit Timestamp (e.g., 0x194131ACC98)
    icon,                # Weather Icon ID (0x00 - 0x10)
    temperature_c,       # Temperature in Celsius
    convert_f,           # Convert to Fahrenheit (0x00 or 0x01)
    time_format,         # 12h/24h Format (0x00 or 0x01)
]
Weather Icon IDs:

ID	Description
0	Nothing
1	Night
2	Clouds
3	Drizzle
4	Heavy Drizzle
5	Rain
6	Heavy Rain
7	Thunder
8	Thunderstorm
9	Snow
10	Mist
11	Fog
12	Sand
13	Squalls
14	Tornado
15	Freezing Rain
16	Sunny
Commands Overview
General Commands
Command 0x02:
Purpose: Potentially related to enabling anti-shake functionality.
Details: BLE_REQ_PUT_ANTI_SHAKE_ENABLE
Command 0x03:
Purpose: Silent mode toggle.
Behavior: Acts as a toggle; the same command enables and disables silent mode.
Command 0x05:
Subcommand 0x02: Sets log level (verified).
Other Subcommands: Enable debug mode, etc.
Command 0x07:
Purpose: Displays a countdown, possibly a precursor to teleprompter functionality.
Command 0x09:
Purpose: Teleprompter functionality.
Command 0x0A:
Purpose: Navigation.
Details: Utilizes RLE compression specifically for navigation.
Commands 0x0D, 0x0E, 0x0F:
Purpose: Translation functionalities.
Details:
0x0D: Initiate translation.
0x0E: Session-related commands.
0x0F: Finalize translation.
Dashboard Commands (0x06)
Purpose: Update dashboard information such as time and weather.
Structure: Detailed in the Packet Structures section.
Additional Details:
Timestamps are sent twice: once in seconds (32-bit) and once in milliseconds (64-bit).
data1 (max length 37) and data2 (max length 42) are included but their specific purposes remain unclear.
Teleprompter (0x09) & Navigation (0x0A)
Teleprompter (0x09):
Purpose: Display teleprompter text or cues.
Current Status: Pending further reverse engineering.
Navigation (0x0A):
Purpose: Handle navigation functionalities.
Compression: Uses RLE compression.
Observations: Navigation-related commands and data are more intricate due to compression.
Translation Commands (0x0D, 0x0E, 0x0F)
Purpose: Facilitate translation features.
Current Findings:
Speech-to-text processing may occur entirely on the glasses.
Initially thought to involve cloud services (e.g., Azure Speech Services), but later findings suggest on-device processing.
Silent Mode (0x03)
Purpose: Toggle silent mode on/off.
Behavior: Single command acts as both enable and disable, requiring state tracking.
Other Notable Commands
Command 0x01:
Usage: Unclear, possibly related to initial synchronization or handshake.
Command 0x26:
Purpose: Sets both distance and height parameters.
Commands for Logging:
Command 0x236C: Enable logs.
Command 0x236C31: Disable logs.
Special Characters:
#r: Restarts the glasses.
#t: Returns debug information.
Notification Whitelisting
Purpose: Control which app notifications are displayed on the glasses.
Implementation:
The glasses receive a whitelist of app IDs for which notifications should be displayed.
Users can control "notification types" by setting specific app IDs:
com.android.phone_missed / com.apple.mobilephone_missed
com.android.phone_incall / com.apple.mobilephone
com.apple.MobileSMS / com.android.even_sms
Limitations:
Maximum of 100 apps can be added to the whitelist.
Reversal Possibility:
Stocks & news notifications can be easily reversed, allowing malicious users to push their own news sources or stock information.
App Interaction & Whitelisting
JSON Usage:
Unlike other system components, certain parts utilize JSON for data handling.
Example: The app list command is defined in the Dart file setup.dart.
App List Command:
Responsible for sending the whitelist of apps to the glasses.
Filters which notifications the glasses display based on the whitelist.
API Interaction:
Firmware updates are fetched from api.evenreal.co.
Potential API endpoints for firmware flashing are based on Flutter-nRF-Connect-Device-Manager.
Image Handling
Bitmap (BMP) Handling
Image Upload:
Images are sent as entire .BMP files rather than raw pixel data.
Attempting to send raw bits (0s and 1s corresponding to pixels) does not render the image on the glasses.
Bitmap Specifications:
Format: Likely 4-bit greyscale, although initial assumptions suggested possible opacity handling.
Slots: Firmware indicates slots for up to 4 images, suggesting limited image storage or predefined slots for specific functionalities.
Compression: RLE compression is specifically used for navigation-related images.
Image Customization:
Possibility to send stereoscopic BMPs to each eye for 3D effects by sending different images to each lens.
Artifacts:
Sending BMPs with lower resolution results in visual artifacts, possibly due to improper scaling or resolution mismatches.
Navigation Map Rendering
RLE Compression:
Utilized exclusively for navigation maps.
Simplifies data transmission by compressing repetitive patterns in map images.
Opacity Handling:
Despite using bitmap images, the navigation app manages varying levels of perceived opacity, potentially through stroke variations or multiple layers.
Logging & Debugging
Enabling Debug Mode
Process:
Set isDebug=true in the Flutter shared preferences.
Double-tap on the newly created version label within the app.
Effects:
Enables the display of both app logs and firmware-generated logs.
Some special options become available in features like translation and quick notes.
Log Prefixes and Sources
Prefix F4:
Indicates logs originating directly from the firmware.
Storage:
Logs are stored within the Flutter shared preferences.
Log Commands:
Command 0x236C: Enables firmware logs.
Command 0x236C31: Disables firmware logs.
Debug Strings in Firmware
Usage:
Extensive debug strings facilitate navigation and other functionalities.
Potentially includes a JavaScript (JS) engine, though this is speculative.
Impact on Battery:
Unclear; concerns exist about the battery consumption impact due to extensive logging.
Synchronization & Communication Issues
Master-Slave Behavior
Startup Process:
Upon startup, one glass acts as the master while the other functions as the slave.
Standalone Chips:
Each temple (side) contains a standalone chip, indicating a decentralized architecture.
Desynchronization Causes
Potential Reasons:
Despite clear communication between glasses, occasional desynchronization occurs.
Implications:
Necessitates sending commands twice—once for each glass—to maintain synchronization.
Command Redundancy
Observation:
Commands need to be sent individually to each glass, even though they communicate with each other.
Reasoning:
Unclear; could be due to master-slave dynamics or firmware limitations.
Development Tools & Resources
Firmware Analysis Tools
IDA Pro: Primary tool for disassembling and analyzing firmware binaries.
Ghidra: Recommended for decompiling firmware; supports aarch64 architectures.
nRF5340 DK with Segger J-Link: Hardware tool suggested for deeper firmware analysis.
Libraries & Repositories
Flutter-nRF-Connect-Device-Manager: GitHub Repository
Even_Glasses Python Library: GitHub Repository
Btlejuice: GitHub Repository – Useful for BLE interaction analysis.
Gadgetbridge: Gadgetbridge Website – Potential tool for interacting with G1 glasses.
Community Resources
Discord Channels: Active discussions and shared resources among reverse engineers.
GitHub Repositories:
Fahrplan by Meyskens – Contains Dart models and other relevant code snippets.
Online Articles:
Even Realities G1 AR Glasses Overview
JB Display Products
Miscellaneous Notes
Temperature Handling:
API Returns: Temperature data is returned in Kelvins.
Glasses Processing: Convert Kelvin to Celsius within the glasses based on user settings; further convert to Fahrenheit if required.
User Interface Customization:
Potential to inject custom functionalities between the app and the glasses, although current limitations exist due to firmware constraints.
Emulation:
Lack of available emulators for G1 glasses hinders development and testing without physical devices.
Button Functionalities:
Ten taps reset all user settings.
Five taps reset the glasses entirely.
Firmware Features:
Ability to override the serial number (S/N) of the glasses.
Storage capabilities within the glasses for multiple fonts and images.
Future Work & Considerations
Comprehensive Command Documentation:
Initiate a GitHub repository or Google Doc to compile and verify undocumented commands.
Image Rendering Enhancements:
Explore the feasibility of sending stereoscopic BMPs for 3D interfaces.
Firmware Decompilation:
Continue efforts using tools like Ghidra and IDA Pro to fully understand firmware functionalities.
Emulator Development:
Investigate the possibility of developing an emulator to facilitate app development without physical devices.
API Endpoint Analysis:
Delve deeper into potential API endpoints used for firmware updates and other functionalities.
Community Collaboration:
Encourage sharing of findings, scripts, and tools within the community to accelerate reverse engineering efforts.
Conclusion
This document serves as a foundational reference for those involved in the reverse engineering of G1 AR glasses. Continuous updates and community collaboration are essential to further unravel the intricacies of the firmware and communication protocols. Contributors are encouraged to share new findings and validate existing hypotheses to enhance the collective understanding of the device's inner workings.
