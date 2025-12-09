# NRF52_Zephyr_BLE_Server
We will implement a BLE server on the nrf52840DK board using Zephyr.

## General description

I have already done a rather detailed explanation of BLE using the WB5 from ST, though, I must say, my understanding at the end of that exploration remained “lacking” at best. Yes, I did manage to turn the ST board into a BLE server and communicate with it using multiple aspects of Bluetooth (notifications, reads and writes), but the quality and reliability of the result was subpar to say the least. It was so bad that it made me start this very string of repos, figuring out the nrf52840 as an alternative and dive into its Zephyr architecture. I must say, the nrf52 is generally considered to be a significant improvement compared to the WB5 for a very good reason, as I will be discussing below.

Word of advice: this repo is rather steep. Only engage if there is already some level of understanding how BLE works.

Word of advice 2: the ST BLE app works fine for these projects, though I have found the Nordic BLE Connect app to be infinitely more stable. I recommend switching to that app.

### Bluetooth recap
To give a short recap, Bluetooth is a family of over the air communication protocols with specific physical layers. They work in the 2 GHz regime, similar to Wi-Fi, but unlike Wi-Fi, the protocol shifts the frequency of the coms regularly within a pre-defined range, increasing security significantly – and, by proxy, decreasing data flow significantly compared to Wi-Fi. Due to this security, paired devices will have an inclusive communication with each other. No tertiary party will be shared the exact sequence of the channel shifting, making BLE hart to spoil or eavesdrop upon.

The frequency range of BLE is sectioned into 40 parallel channels. Out of these 40, 3 of them are used for advertising, while the rest are used for data. The 3 are called “primary channels”, while the rest are called “secondary channels”.

#### Protocols
Bluetooth has multiple protocols, such as Classic, Zigbee or BLE. The difference in protocols means different applications:

-	Portable solutions mainly are using BLE or Bluetooth Low Energy since that is the most efficient of all protocol formats. BLE has low data rate and limited reach. It is used mostly to communicate limited data packages, such as mere data strings. It generally does not have enough width to support streaming between devices (though recent advances in BLE indicate some change in that).
-	Bluetooth Classic is the main streaming protocol. It has a higher data rate and power usage since the client device needs to manage the servers pretty much non-stop. Classic has mostly been optimized for audio streaming through the A2DP (Advanced Audio Distribution Profile) since its first implementation in the early 2000’s.
-	Zigbee is optimized for more mesh networking for IoT applications. It is mostly used in scenarios where the devices are powered externally with no upfront limit on power consumption.

We will be using BLE here just like we did with the WB5. That’s partly due to the demands of my application, partly due to the fact that – unlike the ST device - nrf52 devices do not support Classic or Zigbee. This latter should be kept in mind when designing an architecture using the nrf52.

#### What’s in the Nordic BLE box?
The Nordic training for BLE has a very informative image depicting Bluetooth and how it is built up.

On the base level – which we could call the “hardware level” – we have the Host Controller Interface, which is the physical layer (PHY) of BLE (e.g., antenna, frequency shifting ICs) and the link layer, which physically glues the PHY to the rest of our device. To make a parallel, in the WB5, these hardware elements were the IPCC (a designated communication line between the application core and the coprocessor), HSEM (a set of designated semaphores to allow data exchange and synchronisation between the two cores), RF (antenna driving) and the RTC clock (clocking), plus the actual coprocessor, an M0 separate from our M4 app core. Mind, in the nrf52840 we only have one core, thus the data modulation into BLE packages and the antenna control will be done using an RTOS library called “SoftDevice Controller” – a thread-based driver, implemented into the Zephyr BLE API – and not a secondary coprocessor.

Above the “hardware level”, we will have the “Zephyr Bluetooth Host”, again implemented into the Zephyr BLE API. This host will be the one that defines the Generic Access Profile (GAP), the Generic Attribute Profile (GATT) and the Attribute Protocols (ATT), i.e. the elements that make the protocol work. It also has a designated security manager (SMP) to implement methods of security and a Logical Link Control and Adaptation Protocol (L2CAP) which, in my understanding, will feed the protocol from the application.

The nrf52840 running the Zephyr API has full support to be a peripheral device (or server) or a central device (client).

#### BLE topology blocks
There are two main elements of the BLE protocol that we should mention:

-	GAP is where we define if we are a peripheral or central from a physical point of view. It is also where we set the advertising information (server side) or define the exact manner of seeking out a device to connect to (client side).
-	GATT is what we are using after communication is established. Within GATT is where we define if we are peripheral (server) or central (client) from a topology point of view. In an obvious one-on-one communication, the peripheral device is the server and the central is the client, though this isn’t strictly an obligation: it is perfectly possible to have a device both as a client and a server in more complex networks. Here we will stay with a simple one-on-one communication, thus server/peripheral and client/central can be interchanged. We can calibrate the BLE bus after connection is established without sharing data, which will be the second project. BLE manages data is standardized data formats/structs called “attributes”. The GATT layer is there to organize attributes, organizing and classifying them into profiles, services or characteristics. Manipulating these layers to form services of different types (writes, reads, notifications) will be the third project of this repo.

Regarding the actual structure of a BLE topology, a profile is the highest level struct, followed by the service and the characteristics. A profile can have multiple services but no characteristics. A service can have multiple characteristics while a characteristic can have multiple attributes. There are a myriad of profiles that are already defined on the official Bluetooth website to help with interoperability.

Anyway, we will look into GATT a bit more when we reach the third project.

## Previous relevant projects:
In the previous repos I have done on the WB5 I did most of the introduction for BLE. There I have discussed many things more in detail than how I will do it here. As such, they are very much recommended:

STM32_BLE_Custom_Server_UART Public

STM32_BLE_Custom_Server_Notify Public

STM32_BLE_Custom_Server_Read Public

## To read
I will be going through the BLE course on the Nordic website:

Bluetooth Low Energy Fundamentals - Nordic Developer Academy

Here is the github repo for the course:

GitHub - NordicDeveloperAcademy/bt-fund

Here one can find all the official BLE profiles:

Specifications | Bluetooth® Technology Website

We will be using the Zephyr macros again (but not the nrf52 macros, so search the right documentation for answers):

Zephyr API Documentation: Introduction

## Particularities
With all the general discussion out of the way, let’s take a look into implementing everything.

Mind, we will strictly remain a SERVER within this repo and no client setup will be done.

I will also add BLE to the project sequence I have done finishing with spi, though the exact implementation of BLE should be clear from the code.

### Advertising (BLE Adv test)
I think it is worth considering primary channel communication as a completely different type of coms using the BLE bus compared to secondary…or if not that, then consider the two as as two very different phases of the same coms. I find that to be a good philosophy since, as presented below, it is perfectly possible to use only primary channels for communication. There we simply set out a broadcaster device – technically a BLE beacon – that will ignore any attempt to be connected to. In this case, we can remain on the GAP level and not use GATT at all.

We will do this in the first example project below.

#### What happens on the GAP level?
Advertising is, by its nature, a one-way communication from the server to the client and does not depend on the client doing, well, anything. The server just belches data on the (primary channels of the) bus at regular intervals, waiting for the client to pick up on the advertising and ask for connection, progressing the communication to the next phase using the secondary channels.

Before connecting, the client can send something called a “scan request” and have a “scan response” from the server. This will result in an asynch exchange of data, mostly additional information regarding the server. This will happen still only on the primary channels. In the BLE app, this would be opening up the information board of the server, checking what it the advertises without connecting to it.

#### Setting up the primary channels
Primary channel (GAP) communication has its own parameters and are independent to the actual “connected” BLE communication on the secondary channels (GATT). These parameters are:

-	Advertising interval: The advertising packets are sent out regularly with the advertising interval. The more frequent the advertising is, the more frequently the BLE of the system must be activated, thus increasing the frequency directly leads to higher power consumption.
-	Advertising channels: As mentioned above, the frequency range of BLE is sectioned into 40 parallel channels. Out of these 40, 3 of them are used for advertising, while the rest are used for data. These 3 primary channels are channels 37,38 and 39 and are placed at the two extremes and the centre of the band to limit noise.
-	Scan window and scan interval: this is on the client’s side. It defines, how long a device scans for advertising packets and how often it does so. 
-	Advertising type: we can limit accessibility of a server to certain devices, make the advertising directed to only specific scanners or make the transfer to secondary channels automatically between “known” devices. I won’t be looking into these options and keep everything open.
-	Addressing: like in all networks, BLE devices also have their own addressing that a 48-bit number. This number can be set specifically, though we won’t be looking into it and keep the addressing random.

Advertising itself will be done using advertising packets. In general, a BLE packet is made from 4 elements: a preamble, an access address, a PDU – which is the data section in the packet – and the CRC. The PDU then can be either for advertising or for data. When used for advertising, the PDU is a header and a payload where the header holds some of the information – for instance, the advertising type, the address type and the payload length - necessary to receive the packet and the payload holds the 48-bit BLE address and the data itself. This data then holds the length of the advertisement data structure (AD), the type and the data-data itself (yes, confusing). The data-data will be then the parameters we associated with GAP in the WB5 study, like the device local name or the service UUID (see the WB5 repo for more on that). We also set the advertisement flags here, calibrating the GAP to allow connections or not. In general, as many AD types should be filled up with information as possible since the advertising data tells a lot about the server we are interacting with.

#### Get the syntax sorted
First and foremost, Kconfig must include the “CONFIG_BT=y” to include the BLE stack in our code. This will set the BLE to default configuration, enabling the radio at 0 dbM power and the “softdevice controller” – i.e. the host controller.

Then we must give the BLE device a name using “CONFIG_BT_DEVICE_NAME”. This will be in the advertising data, put into the designated AD.

We need to add the “gap.h” and “Bluetooth.h” header files to the main code.

The advertising data will be an “bt_data” struct that we will have to manually fill.

For general advertising, we will have to define a bt_data called “ad” or similar. This will be what we see when scanning for our device.

First, we will have to sort out the AD flags using the “BT_DATA_BYTES” macro on the “BT_DATA_FLAGS” element in this “ad” struct. The flags must be set to the type advertising we intend to do. The flags can be found in “gap.h”. (Note: Since Nordic devices only supporting BLE, the “BT_LE_AD_NO_BREDR” flag must always be set for the nrf52! Setting the flag will block Bluetooth Classic.)

Afterwards, we need to set the advertising packet payload using the “BT_DATA” macro. This will take the AD type, the AD data pointer and the length of the AD as input. We will have to set all the AD elements here. Once the flags and the payload are set, we should have the “bt_data” struct for the AD loaded.

Scan response comes in when we further investigate the advertised device – i.e. click on the advertising without connecting yet. For that packet, we will have to set an additional “bt_data” struct for that and fill that packet up as well. Mind, if we want to mimic a specific service that exists already, we would need to set the UUID value here.

Once done, we need to enable the BLE stack with “bt_enable”. This function is blocking when calling it with a NULL pointer and must be called with a “bt_ready_cb” callback in any other case.

If bt is enabled, we can start advertising using the “bt_le_adv_start” function. This function will be fed the type of advertising (and the parameters for advertising), the advertising packet, its size, the scan response packet and the scan response packet size. Regarding type, we can set the parameters ourselves or use one of the pre-set ones, like “BT_LE_ADV_NCONN” which will be a non-connectable server with 100 ms advertising interval.

If we wish to not use this pre-set configuration, we need to fill up a “bt_le_adv_param” struct using the “BT_LE_ADV_PARAM” macro and then run the “bt_le_adv_start” function with this “bt_le_adv_param” struct instead of the “BT_LE_ADV_NCONN”.

If we want to add a custom data to the advertising packet, we can do that by assigning it to “BT_DATA_MANUFACTURER_DATA” type when constructing the scan response struct. Dynamically updating the advertising data is done running the “bt_le_adv_update_data” function.

#### Summary
Code section: here we have a simple BLE beacon (server) set to advertise data on the bus. We will be able to see it with any kind of BLE scanning app on our smart phones. When the advertising is further interrogated, we will be able to see what we have put into the “scan” value, i.e. an arbitrary url, a random UUID number plus a custom data packet counting button presses on btn1. The custom data element is updated after the button press flag has been updated. (Note: not having the BLE activity running in a separate thread or a work queue may potentially lead to timing issues and the BLE collapsing on itself. Best practice to remove it from the main loop to a separate thread.)

Board type: custom board v7

### Connection baseline (BLE_Conn_test)
As mentioned above, after the advertising has been done and the client has found the server, we “switch” to connection using the secondary channels. This is an obvious benefit compared to staying on the primary channels since we have the rest of the 40 channels available for transfer plus a plethora of security options. 

In general, connecting between devices is done by the client sending a connection request PDU packet. The server waits for a few milliseconds for such packet after every advertising packet. Disconnecting is done when the client sends a termination packet or if packets are not transferred regularly on the bus between the components and a “connection supervision timeout” has been reached by either the server or the client.

Normally, advertising packets hold all the necessary information to initiate a connection and progress to the secondary channels. Nevertheless, the connection will have a separate set of parameters, ones that will be shared between the client and the server in order to properly manage the communication. The parameters must be shared and matched between components, like it would be on any other communication bus. On startup, the parameters are standard values, widely considered to be compatible with any server and client.

The PDU packets for the connection are similar to advertising PDU packets, where we have a 2-byte header, followed by a flexible sized payload section. The payload section is then divided up into the L2CAP header, the ATT header and the ATT data.

Here in the second project, we will connect to the advertising server and then set up the BLE bus according to the parameters we wish. Mind, we are still not using any services here, just setting the bus up so the GATT library won’t need to be included.

#### Connection parameters
Most of these parameters are set by the client within the connection request packet, albeit the server can ask for changes. They all relate to the physical parameters of the bus and the communication that is done on said bus. Changing these parameters allows for the optimisation of the connection. 

The parameters are the following:

-	Connection interval: The connection interval is what defines, how often the devices on the bus are communicating with each other. This is what makes BLE really low energy, since both devices – or their Bluetooth elements at least – can go to sleep between these intervals.
-	Supervision timeout: This is the agreed timeout between devices. If it is reached, the devices disconnect from the bus.
-	Latency: tells the server to skip certain events if it has no data to send over. Allows saving power since the device remains idle without telling about it to the client.
-	PHY: Setting the data rate of the signal. Mind, we can change this to 2M PHY, which would double the data rate, decrease power consumption and decrease the range of our communication. We can also use a “coded PHY”, which will add coding schemes to correct packet errors, but decrease the data rate. The standard PHY value is 1M.
-	Data length: We can limit the maximum physical packet length in one payload and we can limit the number of bytes within a specific GATT operation (MTU). In practical sense this means that we can limit how much data we send over during one physical packet transfer and during one GATT operation. Mind, an operation packet will be cut it into smaller chunks if the payload is smaller than the operation’s limit. Ideally, the packet size is set to be greater than the operation’s limit, meaning that the transfer occurs using one physical packet only. Each physical packets will have a L2CAP and an Attribute header, thus the less physical packets we generate, the more deadweight header information we will not have to transfer. The standard MTU value is 23 bytes and the payload is 27 bytes.

We won’t set the MTU here in this project but the next. I will explain it there, why.

#### Connection syntax
First and foremost, we will add the “conn.h” library to our code.

It is good practice to define an arbitrary (not Zephyr-based) struct handle for our connection which will be a “bt_conn” struct. We will be referring back to our connection through this handle, i.e., it will store its properties. These properties are whatever we want them to be. (Mind, here I have commented out referring to it since we only have one connection and thus do not need it.)

The most basic connectable discovery mode is “BT_LE_AD_GENERAL”, that must be added to the discovery mode GAP flag in advertising to allow connection to the server (i.e. go back to GAP and add it). We also have to set the GAP parameter from “BT_LE_ADV_OPT_NONE” to “BT_LE_ADV_OPT_CONN” so we won’t have a beacon anymore but a connectable server.

We also need to add “CONFIG_BT_PERIPHERAL=y” to define our device as a server.

Once engaged, the BLE API will “communicate” with our code then using callbacks. It is necessary to set a callback up for each action we expect the API to execute, such as connecting or disconnecting There actually are a lot of options here, consult the documentation. The entire list of “bt_conn_cb” callbacks are in the Zephyr documentation.

Anyway, the “bt_conn_cb” struct will track the state of the connection for us and by proxy call the callbacks in case an event has occurred. Mind, a “connected callback event” is not the same as a “connection event”, i.e. these are callback events related to having the devices being already connected to each other. This struct is attached to the BLE connection using the “bt_conn_sb_register” function. (Alternatively, the “BT_CONN_CB_DEFINE” macro can be used to do the two steps above in one swoop.) If “bt_conn” is used, we can use the “bt_conn_ref” in the “connected” callback and the “bt_conn_unref” in the disconnect callback to update the state of our connection for future reference (here this is not necessary).

Mind, there is the “recycle” callback as well that is recommended to be defined. This will technically be the restart of advertising in case the API encounters a failure. Recycling is asynch to the code execution and thus should be executed in a work queue or a separate thread to avoid timing-related coms collapse. A recycle event does not occur upon BLE being enabled, thus advertising should be started “manually” on first startup. (I am not implementing recycling here to avoid overcomplicating the repo with RTOS. This does mean though that the server will not be able to recover after a disconnection.)

Once we have these set, we will be able to physically connect to our server (and do little else since don’t have GATT added yet).

#### Parameter setting syntax
Now that we are connected, how do we set the connection parameters with the server? We already know that on every connection, the client will start communicating with the server using a standard value for all the parameters mentioned above, but these may not be what we want to use.

If we want to know, what the pre-set parameters are, we can put into the “connected” callback a simple logging of these standard values. The parameters will be stored in a “bt_conn_info” struct that can be populated by the existing parameters using the “bt_conn_get_info” function and then printed out on the terminal. Alternatively, we can just use the “parameter update” callback from “bt_conn_cb” instead to print out the same values every time a parameter has changed.

Anyway, parameters themselves are hard-baked into the server device through KConfig and normally are not changed on the fly. Instead, what we are doing is requesting a regular update on the parameters by setting the “CONFIG_BT_GAP_AUTO_UPDATE_CONN_PARAMS=y”. This will make our server check the parameters of the coms and automatically request an adjustment from the client if there is a discrepancy. This is helpful for recovering the bus after a disconnect. If we don’t want a regular update, an update can be demanded by the server running the “bt_conn_le_param_update” function. The parameters we handle this way are the latency, the timeout and the interval.

Regarding the PHY, we will always start with a modulation 1M PHY, which will be a 1 Megabit transfer. First and foremost, “CONFIG_BT_USER_PHY_UPDATE=y” should be added to KConfig. The PHY update should then occur within the “connection” callback by calling the “bt_conn_le_phy_update” function on a “bt_conn_le_phy_param” struct that will hold the Rx/Tx parameters and the new communication option (so this is not in KConfig). The “bt_conn_cb” can have a callback on PHY update should we wish to implement it.

Regarding the payload data length, it follows the same philosophy as the PHY:  define a function and call it within the “connected” callback. We do need to enable the data length update in KConfig (“CONFIG_BT_USER_DATA_LEN_UPDATE=y”) and make the Tx/Rx buffers adequately big also in KConfig, otherwise the build will fail so here we use KConfig as well as an update function.

#### Summary
Code section: here we are changing our beacon into a connectable server device. Our server then requests changes to the BLE bus parameters to fit our needs. We have callbacks attached to each change that will print out the changed values. Mind, we will need to be connected to our device to see any of these printouts take effect since they are all dependant on an actual connection to a client. Of note, we are NOT including GATT libraries in our code yet, thus we aren’t changing the MTU yet (the data length of a GATT transfer).

Board type: custom board v8

### Data transfer (BLE_GATT_test)
Now that we have a working BLE connection, we will implement GATT to the code and recreate the write/notify/read trio we have done for the WB5. To be more precise, what we want is:

- the device writes to the terminal whatever we send it from our phone
- send a counter regularly to the client when the notification is enabled
- send data from the BMP280 upon a readout request

Below I will implement all three.

#### What the GATT?
The GATT layer is where we fill up our data packets with the communication architecture that we will be using. As mentioned above, we will have profiles, which will have services which will have characteristics. (I won’t be defining profiles in this project.)
The way we set services will define the type of data access we will have using the BLE bus. We can have client-initiated operations – read, write (with acknowledge), write (without acknowledge) – and server-initiated operations – notify and indicate (a one-off notify). Mind, server operations must still be enabled by the client first, albeit afterwards, they will be executed independent of the client.

All operations are to be considered within the framing of a server, meaning that reading/writing/notifying occurs with the characteristics of the service and NOT with the memory element containing the data. In other words, every value we would be interacting with in our server must be first imported/exported in to/out of the service. Luckily macros and functions will do this for us automatically.

The services are organised along ATT layers where ATT attributes define how data is stored and accessed by our server. An ATT attribute always consists of an ATT handle (a hex number), a UUID, a permission and a value or user data.

A service has a designated ATT to describe it (called service declaration attribute) while the characteristics within the service will also have their own ATT elements (called characteristic definition attributes).

The service declaration ATT has the service ATT handle, a UUID of 0x2800 – “hey, this is a service here” UUID – a permission value than can be set to our needs (though it should be read-only if we want the service to remain unchanged) and a value which will be the designated UUID for the particular service we are declaring (for example, the official heart rate service has the UUID of 0x180D).

Each service may have none or multiple characteristics, each with its own definition ATTs. The characteristic must also have its own declaration ATT like the service, followed by the ATT for the value of the characteristic. The characteristic’s declaration will have the handle for the characteristic, followed by a UUID of 0x2803 – “hey, this is a characteristic here” UUID – the permission (which should be read-only to maintain the characteristic as-is) and the properties/value handle/UUID of the characteristic. Within the ATT for the characteristic’s value itself, the handle and the UUID must be the same as the ones we have given in the declaration’s value section, followed by the permission (which can be anything and will only apply to the characteristic’s ATT value’s value section) and the user data. (Of note, an ATT has a section called “value” and a characteristic has also a value, which by itself is an ATT. As such, data stored in a service is stored within the characteristic’s value ATT’s value section. Confusing? Yes…) A characteristic may have additional ATTs called descriptors which can give the client options to change the characteristic. Such descriptor is the client characteristic configuration descriptor (CCCD) which will add a “switch” to the characteristic to enable it by a client, i.e. to have a notification. The CCCD has a UUID of 0x2902 with read/write permission and a two-bit value to set the service as a notification or an indication.

(Here I suggest looking at lesson 4 “Attribute table” in the “Nordic BLE fundamentals” course. That should give a perfect example of what we are dealing with.)

#### Service ATTs
In order to recreate the three services we had on the wb5, we will need to set the following within the code:

- The uart bridge service declaration will be handle 0x0001 with UUID 0x2800, read only and a random value (service UUID) we wish. The uart bridge characteristic will have handle 0x0002, UUID 0x2803, read-only and the value write/0x0003/random UUID no. 2. The uart characteristic’s value ATT will be then 0x0003/ random UUID no. 2/write/pointer to uart buffer.
- The counter service declaration will be handle 0x0004 with UUID 0x2800, read only and a random UUID no. 3. The counter characteristic will have handle 0x0005, UUID 0x2803, read-only and the value notify/0x0006/random UUID no. 4. The counter characteristic’s value ATT will be then 0x0006/ random UUID no. 4/none/pointer to counter. We will then need to add a CCCD descriptor here as 0x0007/0x2902/read&write/notify (which will be 0x01).
- The sensor readout service declaration will be handle 0x0008 with UUID 0x2800, read only and a UUID no. 5. The readout characteristic will have handle 0x0009, UUID 0x2803, read-only and the value read/0x000A/random UUID no. 6. The readout characteristic’s value ATT will be then 0x000A/ random UUID no. 6/read/pointer to sensor data.

Of note, alternatively, we could just define one service with three different characteristics, but I have decided instead to stay with what I did with the WB5 and not simplify things.

#### MTU setup syntax
Before we get into GATT, we do need to set the last connection parameter – the MTU - up from the previous project. Since the MTU is a GATT property (we didn’t need GATT for anything set above), we couldn’t do it there.
We add the “gatt.h” library to the source code and enable the GATT library in KConfig using” CONFIG_BT_GATT_CLIENT=y”.

Compared to the other parameters, setting the MTU is a bit more complex. We will need to call the function “bt_gatt_exchange_mtu” on the connection handle and a “bt_gatt_exchange_params” struct. This struct will have a callback function that we will need to set ourselves – in the code, it will be a simple printout of the MTU value. The updated values for the MTU itself will be extracted from KConfig (add ”CONFIG_BT_L2CAP_TX_MTU=247”). The mtu update will then need to be put into the connection callback, like the PHY and the payload length.

#### GATT syntax
First, we will need to add the “uuid.h” library and define all the random generated UUIDs by feeding values to the “BT_UUID_128_ENCODE” macro and then declaring each UUID using the “BT_UUID_DECLARE_128” macro. Here that would be 6 different UUIDs.
Then we need to define the services by their names and their defining UUIDs using the “BT_GATT_SERVICE_DEFINE” macro and the “BT_GATT_PRIMARY_SERVICE” macro.

The “BT_GATT_CHARACTERISTIC” macro is then there to attach the characteristics to the services. This macro will take callback functions for both read and write services, where we will be adding the manipulation – i.e. writing to or readout – of the attribute and the underlying variables into these callback functions.

For write, we merely need to manage to callback function. The callback must follow the topology of “bt_gatt_attr_write_func_t”. The buffer with the incoming data elements will be stored in the assigned “buf”. We can simply take it in the callback and print it out on the terminal.

For the readout, the characteristic definition macro from above will take in the pointer for the variable that the characteristic’s value should hold. The readout callback function is “bt_gatt_attr_read_func_t” type and will have to include the “bt_gatt_attr_read” to send the readout data to the client. This “sending” function will take every element from the callback except the value and the “len” for the value which we are not stored within the callback and thus we must set it ourselves. These will be a pointer to the variable we have defined above in the definition as the value/user data for the characteristic well as the size of the variable. As a best practice, it is best to point to the user data section of the characteristic albeit we could forego that point to the variable instead, the two are the same. Mind, within this callback we will have to update the user data variable we have given in the definition macro.

Lastly, we will set a notification. The characteristic will be for a new random UUID, a notification type, “none” as permission and NULL for the callbacks and the readout variable. We will have to add a descriptor though with a callback function and read/write permissions. The descriptor callback will simply extract the notification’s on/off switch from the stack so we may use it in a separate function to then run the “bt_gatt_notify” function when the switch is on. The “bt_gatt_notify” will insert the new data value into the attribute of the notification. Mind, the notification refresh rate will depend on the time intervals with which we call the “bt_gatt_notify” function.

Of note, a real notification must run in a separate thread to avoid collapsing the stack. I didn’t implement that to avoid getting into RTOS territory. Instead, we will do a notification once every time we notify the server. This “should” mostly avoids the potential timing issue of updating the notification regularly to the point where I have not seen the BLE stack collapsing.

#### Summary
Code section: The code will set up a read service, a write service and a notification. The read service will send over the attached BMP280 sensor’s ID upon readout. Write is an uart bridge, printing out the message we type into the phone byte after byte. Notify will increment a counter every time we activate the notification.

Board type: custom board v9, ((Adafruit Feather M0 as the other board for uart)

#### Final notes
We are using logging to print out when using the callbacks since logging runs in a separate thread and won’t get jumbled up by timing. Printk will not work once we have the BLE stack running.

Our services and characteristics will all be called “unknown” when investigating it with our phones. The reason for that is that we are running them using random UUID numbers, not official Bluetooth ones. Mind, if we change the UUIDs to “known” values – see Bluetooth website - then the service will be recognised and named.

I have removed the “fake” UUID we were publishing in the advertising section when implementing GATT in order to have a clear list of services running on our server.

Throughout all the definitions used in the GATT coding, we can use “BT_GATT_ERR” with the respective error byte to indicate an error to the API.

### Board versions
The new board versions are the following:

-	v7: added BLE advertising
-	v8: added BLE bus configuration
-	v9: added BLE services

We were exclusively working in the defconfig file of the board and left the rest of the devtree untouched compared to board version v6. As such, one can use board version v6 with an updated proj.conf instead to reach to the same board definition as board version v9.

## User guide
A short description on what is going on in each project elements was provided above.

## Conclusion
We have successfully recreated the WB5 projects using the nrf52. I must say, I am significantly more satisfied with the outcome than what I have managed to churn out of the WB5.

Next, we will take a look into setting the nrf52 up as BLE client instead of a server.

