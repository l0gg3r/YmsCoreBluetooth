# YmsCoreBluetooth v0.942 (beta)
A framework for building Bluetooth 4.0 Low Energy (aka Smart or LE) iOS or OS X applications using the CoreBluetooth API. Includes *Deanna* and *DeannaMac*, applications to communicate with a [TI SensorTag](http://processors.wiki.ti.com/index.php/Bluetooth_SensorTag) for iOS and OS X respectively.

* [YmsCoreBluetooth API Reference](http://kickingvegas.github.io/YmsCoreBluetooth/appledoc/index.html)

## YmsCoreBluetooth Design Intent: Or Why You Want To Use This Framework
### tl;dr 
* ObjectiveC Block-based API for Bluetooth LE communication.
* Operations (e.g. scanning, retrieval, connection, reads, writes) map to the data object hierarchy of CoreBluetooth.
* You can build apps using CoreBluetooth [faster](http://yummymelon.com/ymsblog/sensortag-remote-control-for-itunes.html).

### A More Detailed Explanation

#### Blocks are cool.
Transactions in Bluetooth LE (BLE) are two-phase (request-response) in nature: CoreBluetooth abstracts this protocol so that request behavior is separated from response behavior. The two phases are reconciled using a delegation pattern: the object initiating the request phase has a delegate object with a delegate method to handle the corresponding response phase. While functional, the delegation pattern can be cumbersome to use because the behavior for a two-phase transaction is split into two different locations in code.

A more convenient programming pattern is to use a callback block which is defined with the request. When the response is received, the callback block can be executed to handle it. *The design intent of YmsCoreBluetooth is use Objective-C blocks to define response behavior to BLE requests*. Such requests include:

* scanning and/or retrieving peripheral(s)
* connecting to a peripheral
* discovering a peripheral's services
* discovering a service's characteristics
* write and read of a characteristic
* setting the notification status of a characteristic

#### Hierarchical operations are cool.
The data object hierachy of CoreBluetooth can be described as such:

* A CBCentralManager instance can connect to multiple CBPeripheral instances.
    * A CBPeripheral instance can have multiple CBService instances.
	    * A CBService instance can have multiple CBCharacteristic instances.
		    * A CBCharacteristic instance can have multiple CBDescriptor instances.
			
However the existing CoreBluetooth API does not map BLE requests to the data object hierarchy. For example connection to a CBPeripheral instance is accomplished from a CBCentralManager instance instead of from a CBPeripheral. Writes, reads, and setting the notification state of a CBCharacteristic are issued from a CBPeripheral instance, instead of from CBCharacteristic. *YmsCoreBluetooth provides an API that more naturally maps operations to the data object hierarchy*.

YMSCoreBluetooth defines container classes which map to the CoreBluetooth object hierarchy:

* YMSCBCentralManager - Contains a CBCentralManager instance.
    * YMSCBPeripheral - Contains a CBPeripheral instance.
        * YMSCBService - Contains a CBService instance.
             * YMSCBCharacteristic - Contains a CBCharacteristic instance.
			     * YMSCBDescriptor - Contains a CBDescriptor instance.

However, they differ from CoreBluetooth in that operations are done with respect to the object type:

* YMSCBCentralManager
  - scans for peripherals
  - retrieves peripherals
* YMSCBPeripheral
  - connects and disconnects to central
  - discovers services associated with this peripheral
* YMSCBService
  - discovers characteristics associated with this service
  - handles notification updates for characteristics which are set to notify
* YMSCBCharacteristic
  - set notification status (on, off)
  - write value to characteristic
  - read value of characteristic
  - discover descriptors associated with this characteristic 
* YMSCBDescriptor
  - write value to descriptor
  - read value of descriptor
  
### Show Code
#### Scanning for Peripherals

In the following code sample, `self` is an instance of a subclass of YMSCBCentralManager.

    __weak DEACentralManager *this = self;
    [self scanForPeripheralsWithServices:nil
                                 options:options
                               withBlock:^(CBPeripheral *peripheral, NSDictionary *advertisementData, NSNumber *RSSI, NSError *error) {
                                   if (error) {
                                       NSLog(@"Something bad happened with scanForPeripheralWithServices:options:withBlock:");
                                       return;
                                   }
                                   
                                   NSLog(@"DISCOVERED: %@, %@, %@ db", peripheral, peripheral.name, RSSI);
                                   [this handleFoundPeripheral:peripheral];
                               }];

#### Retrieving Peripherals

In the following code sample, `self` is an instance of a subclass of YMSCBCentralManager.

	__weak DEACentralManager *this = self;
	[self retrievePeripherals:peripheralUUIDs
					withBlock:^(CBPeripheral *peripheral) {
						[this handleFoundPeripheral:peripheral];
					}];
  
  
#### Connecting to a Peripheral

In the following code sample, `self` is an instance of a subclass of YMSCBPeripheral. Note that in the callbacks, discovering services and characteristics are handled in a nested fashion:

	- (void)connect {
		// Watchdog aware method
		[self resetWatchdog];

		[self connectWithOptions:nil withBlock:^(YMSCBPeripheral *yp, NSError *error) {
			if (error) {
				return;
			}

			[yp discoverServices:[yp services] withBlock:^(NSArray *yservices, NSError *error) {
				if (error) {
					return;
				}

				for (YMSCBService *service in yservices) {
					if ([service.name isEqualToString:@"simplekeys"]) {
						__weak DEASimpleKeysService *thisService = (DEASimpleKeysService *)service;
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError *error) {
							[thisService turnOn];
						}];

					} else if ([service.name isEqualToString:@"devinfo"]) {
						__weak DEADeviceInfoService *thisService = (DEADeviceInfoService *)service;
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError *error) {
							[thisService readDeviceInfo];
						}];

					} else {
						__weak DEABaseService *thisService = (DEABaseService *)service;
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError *error) {
							for (NSString *key in chDict) {
								YMSCBCharacteristic *ct = chDict[key];
								//NSLog(@"%@ %@ %@", ct, ct.cbCharacteristic, ct.uuid);

								[ct discoverDescriptorsWithBlock:^(NSArray *ydescriptors, NSError *error) {
									if (error) {
										return;
									}
									for (YMSCBDescriptor *yd in ydescriptors) {
										NSLog(@"Descriptor: %@ %@ %@", thisService.name, yd.UUID, yd.cbDescriptor);
									}
								}];
							}
						}];
					}
				}
			}];
		}];
	}


#### Read a Characteristic

In the following code sample, `self` is an instance of a subclass of YMSCBService. All discovered characteristics are stored in [YMSCBService characteristicDict].

	- (void)readDeviceInfo {

		YMSCBCharacteristic *system_idCt = self.characteristicDict[@"system_id"];
		__weak DEADeviceInfoService *this = self;
		[system_idCt readValueWithBlock:^(NSData *data, NSError *error) {
			NSMutableString *tmpString = [NSMutableString stringWithFormat:@""];
			unsigned char bytes[data.length];
			[data getBytes:bytes];
			for (int ii = (int)data.length; ii >= 0;ii--) {
				[tmpString appendFormat:@"%02hhx",bytes[ii]];
				if (ii) {
					[tmpString appendFormat:@":"];
				}
			}

			NSLog(@"system id: %@", tmpString);
			_YMS_PERFORM_ON_MAIN_THREAD(^{
				this.system_id = tmpString;
			});

		}];

		YMSCBCharacteristic *model_numberCt = self.characteristicDict[@"model_number"];
		[model_numberCt readValueWithBlock:^(NSData *data, NSError *error) {
			if (error) {
				NSLog(@"ERROR: %@", error);
				return;
			}

			NSString *payload = [[NSString alloc] initWithData:data encoding:NSStringEncodingConversionAllowLossy];
			NSLog(@"model number: %@", payload);
			_YMS_PERFORM_ON_MAIN_THREAD(^{
				this.model_number = payload;
			});
		}];
	}


#### Write to a Characteristic

In the following code sample, `self` is an instance of a subclass of YMSCBService. In this example, a write to the 'config' characteristic followed by a read of the 'calibration' characteristic is done.

	- (void)requestCalibration {
		if (self.isCalibrating == NO) {

			__weak DEABarometerService *this = self;

			YMSCBCharacteristic *configCt = self.characteristicDict[@"config"];
			[configCt writeByte:0x2 withBlock:^(NSError *error) {
				if (error) {
					NSLog(@"ERROR: write request to barometer config to start calibration failed.");
					return;
				}

				YMSCBCharacteristic *calibrationCt = this.characteristicDict[@"calibration"];
				[calibrationCt readValueWithBlock:^(NSData *data, NSError *error) {
					if (error) {
						NSLog(@"ERROR: read request to barometer calibration failed.");
						return;
					}

					this.isCalibrating = NO;
					char val[data.length];
					[data getBytes:&val length:data.length];

					int i = 0;
					while (i < data.length) {
						uint16_t lo = val[i];
						uint16_t hi = val[i+1];
						uint16_t cx = ((lo & 0xff)| ((hi << 8) & 0xff00));
						int index = i/2 + 1;

						if (index == 1) self.c1 = cx;
						else if (index == 2) this.c2 = cx;
						else if (index == 3) this.c3 = cx;
						else if (index == 4) this.c4 = cx;
						else if (index == 5) this.c5 = cx;
						else if (index == 6) this.c6 = cx;
						else if (index == 7) this.c7 = cx;
						else if (index == 8) this.c8 = cx;

						i = i + 2;
					}

					this.isCalibrating = YES;
					this.isCalibrated = YES;

				}];
			}];

		}
	}


### Handling Characteristic Notification Updates

One place where YmsCoreBluetooth does *not* use blocks to handle BLE responses is with characteristic notification updates. The reason for this is because such updates are asynchronous and non-deterministic. As such the handler method `[YMSCBService notifyCharacteristicHandler:error:]` must be implemented for any subclass of YMSCBService to handle such updates.

In the following code sample, `self` is an instance of a subclass of YMSCBService.

	- (void)turnOn {
		__weak DEABaseService *this = self;

		YMSCBCharacteristic *configCt = self.characteristicDict[@"config"];
		[configCt writeByte:0x1 withBlock:^(NSError *error) {
			if (error) {
				NSLog(@"ERROR: %@", error);
				return;
			}

			NSLog(@"TURNED ON: %@", this.name);
		}];

		YMSCBCharacteristic *dataCt = self.characteristicDict[@"data"];
		[dataCt setNotifyValue:YES withBlock:^(NSError *error) {
			NSLog(@"Data notification for %@ on", this.name);
		}];


		_YMS_PERFORM_ON_MAIN_THREAD(^{
			this.isOn = YES;
		});
	}

	- (void)notifyCharacteristicHandler:(YMSCBCharacteristic *)yc error:(NSError *)error {
		if (error) {
			return;
		}

		if ([yc.name isEqualToString:@"data"]) {
			NSData *data = yc.cbCharacteristic.value;

			char val[data.length];
			[data getBytes:&val length:data.length];

			int16_t v0 = val[0];
			int16_t v1 = val[1];
			int16_t v2 = val[2];
			int16_t v3 = val[3];

			int16_t amb = ((v2 & 0xff)| ((v3 << 8) & 0xff00));
			int16_t objT = ((v0 & 0xff)| ((v1 << 8) & 0xff00));

			double tempAmb = calcTmpLocal(amb);

			__weak DEATemperatureService *this = self;
			_YMS_PERFORM_ON_MAIN_THREAD(^{
				this.ambientTemp = @(tempAmb);
				this.objectTemp = @(calcTmpTarget(objT, tempAmb));
			});
		}
	}


### Block Callback Design

The callback pattern used by YmsCoreBluetooth uses a single callback to handle both successfull and failed responses. This is accomplished by including an `error` parameter. If an `error` object is not `nil`, then behavior to handle the failure can be implemented. Otherwise behavior to handle success is implemented.

	^(NSError *error) {
	   if (error) {
		  // Code to handle failure
		  return;
	   }
	   // Code to handle success
	}


## File Naming Conventions
The YmsCoreBluetooth framework is the set of files prefixed with `YMSCB` located in the directory `YmsCoreBluetooth`.

The files for the iOS application **Deanna** are prefixed with `DEA` and are located in the directory `Deanna`.

The files for the OS X application **DeannaMac** are prefixed with `DEM` and are located in the directory `DeannaMac`.

## Recommended Code Walk Through

To better understand how YmsCoreBluetooth works, it is recommended to first read the source of the following BLE service implementations:

* DEAAccelerometerService, DEABarometerService, DEAGyroscopeService, DEAHumidityService, DEAMagnetometerService, DEASimpleKeysService, DEATemperatureService, DEADeviceInfoService

Then the BLE peripheral implementation of the TI SensorTag:

* DEASensorTag

Then the application service which manages all known peripherals:

* DEACentralManager

The [Class Hierarchy](http://kickingvegas.github.io/YmsCoreBluetooth/appledoc/hierarchy.html) is very instructive in showing the relationship of the above classes to the YmsCoreBluetooth framework.


## Writing your own Bluetooth LE service with YmsCoreBluetooth

Learn how to write your own Bluetooth LE service by reading the example of how its done for the TI SensorTag in the [Tutorial](http://kickingvegas.github.io/YmsCoreBluetooth/appledoc/docs/tutorial/Tutorial.html).

## Questions

While quite functional, YmsCoreBluetooth is still very much in an early state and there's always room for improvement. Please submit any questions or [issues to the GitHub project for YmsCoreBluetooth](https://github.com/kickingvegas/YmsCoreBluetooth/issues?labels=&milestone=&page=1&state=open).


## Notes

Code tested on:

* iPhone 4S, iOS 6.1.3
* TI SensorTag firmware 1.2, 1.3.
* iMac 27 Mid-2010, OS X 10.8.3

## Known Issues
* No support is offered for the iOS simulator due to the instability of the CoreBluetooth implementation on it. Use this code only on iOS hardware that supports CoreBluetooth. Given that Apple does not provide technical support for CoreBluetooth behavior on the iOS simulator [TN2295](http://developer.apple.com/library/ios/#technotes/tn2295/_index.html), I feel this is a reasonable position to take. Hopefully in time the iOS simulator will exhibit better CoreBluetooth fidelity.

## Latest Changes

### Sat Jun 22 2013 - Interim Release (ver 0.942)
* Reimplemented background thread UI updates to use GCD `dispatch_async()` instead of `performSelectorOnMainThread:` (Issue #61).
* Updated documentation.

### Wed Jun 19 2013 - Interim Release (ver 0.941)
* Support for background thread operation of BLE transactions (Issues #57, #58, #59)

Prior releases ran all BLE transactions off the main UI thread. For a small number of BLE devices (< 5), performance degradation was neglible. However, in an environment with many devices (> 30), support for background operation that does not block the main UI is necessary. Changes so that messages sent to the delegate for `YMSCBCentralManager` and `YMSCBPeripheral` are executed in the main thread.

Similarly, any change to a property of a subclass of `YMSCBService` should be executed in the main thread so that it can be properly key-value observed (KVO) by any UI components.

* API Change: Removed RSSI auto-update functionality from YmsCoreBluetooth.

This directly attributed with background thread operation of BLE transactions. Prior code, if run using a background thread, would not correctly schedule a read of the RSSI value of a peripheral. It is now the responsibility of an application using YmsCoreBluetooth to invoke `readRSSI` and handle the response to update the UI properly.

* Update of Deanna and DeannaMac to support background thread BLE transactions.

#### API Changes

* `[YMSCBPeripheral initWithPeripheral:central:baseHi:baseLo:updateRSSI:]` is obsolete.<br/>Replacing this call is `[YMSCBPeripheral initWithPeripheral:central:baseHi:baseLo:]`

* `[YMSCBPeripheral updateRSSI]` is obsolete.

* `[YMSCBCentralManager initWithKnownPeripheralNames:queue:]` is obsolete.<br/>Replacing this call is `[YMSCBCentralManager initWithKnownPeripheralNames:queue:delegate:]`

* `[YMSCBCentralManager initWithKnownPeripheralNames:queue:useStoredPeripherals:]` is obsolete.<br/>Replacing this call is `[YMSCBCentralManager initWithKnownPeripheralNames:queue:userStoredPeripherals:delegate:]`


### Mon Jun 3 2013 - Disco Release (ver 0.94)
* Issue #9 - OS X Support

YmsCoreBluetooth now supports OS X! Includes app target DeannaMac which replicates functionality found in Deanna for iOS.

Tested Mac Environment:

* OS X 10.8.3
* Cirago Bluetooth 4.0 USB Adapter
* iMac 27 Mid-2010


View [Prior Change List](https://github.com/kickingvegas/YmsCoreBluetooth/blob/master/CHANGES.md)


### License

    Copyright 2013 Yummy Melon Software LLC

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

    Author: Charles Y. Choi <charles.choi@yummymelon.com>





