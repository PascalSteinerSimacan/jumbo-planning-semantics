# Quintiq SC mapping 


## 1. Planning input
Quintiq provides a xsd file for their planning files. If they wish to make changes or add additional fields, then it must be present in the xsd file. We then generate validation code from the xsd [here](https://github.com/simacan/planning-import-xml-schemas). This is then used in planning-receive(parsing) and type definitions for quintiq_planning_service.


## 1.1 Preprocessing

A planning file can have multiple `quintiq:Route`. A `quintiq:Route` can either be a Preload, a PostUnload, both or a MainRoute. This is specified by the xml fields `isPreload` and `isPostUnload`. There MUST be at least one MainRoute. Each route can have multiple `simacan:Trips` . 
For Preload and PostUnload routes, we do not send or show the stops in Unity. Only the stops of the "MainRoute". But we copy the Pickup and Delivery actions from Pre-PostUnload routes and replace the actions of the first or last stop on the main route with them.
Additionally (only for MainRoute), if there are multiple trips within a Route (e.g. TripA & TripB), then these trips need to be "connected". 
This is achieved by taking the first stop of TripB and adding it as the last stop of TripA. But the consignments are only processed once!
This create what is refered to as a "roundtrip".

## 1.2 Rough mapping logic
After the preprocessing we can (roughly) map the following:
 - `quintiq:Stop` -> `simacan:Stop`
 - `quintiq:Action` -> `simacan:Action`
 - `quintiq:ShipmentInfo` -> `simacan:Consignment`
 - `quintiq:ExternalOrder` -> `simacan:LoadCarrier`

## 1.3 Mapping trip level

| SCT Trip field           | XML fields involved  | mapping description 
|--------------------------|----------------------|-----------------------------------------|
| uuid                     |`<xs:element name='TripID' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>`| TripId is extracted from Actions level. [private def parseAsUuid(namespace: UUID, shipper: Shipper, identifier: String) =](https://github.com/simacan/planning-service/blob/5b0a0ba520818d575e7eea0b2391408b8d0a76c9/modules/core/src/main/scala/com/simacan/planning/service/sct/CreateSctUuid.scala#L62-L66) |
| identifier               | `<xs:element name='TripID' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>` | No mapping conversion|
| name                     |`<xs:element name='TripID' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>` | No mapping conversion|
| tripDate                 |```<xs:element name='StartTime' type='xs:dateTime' nillable='false' minOccurs='1' maxOccurs='1'/>```| Child of `<xs:element name="Attributes" minOccurs='0' maxOccurs='1'>`. <br>  No mapping conversion |
| groupingAddressLocation  | `<xs:element name='AddressID' type='xs:string' nillable='true' minOccurs='0' maxOccurs='1'/>`, <br>  `<xs:element name='AddressGroupID' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | Takes the address UUID of the first stop. Address UUID is determined by AdressID or AddressGroupID field (first field takes precedence)  |
| department               |  `<xs:element name='AddressID' type='xs:string' nillable='true' minOccurs='0' maxOccurs='1'/>`, <br>  `<xs:element name='AddressGroupID' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | Same as above but takes the string "name" property of the groupingAddressLocation  |
| routeNumber              | `<xs:element name='TripID' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>` | Same as above  |
| requestedVehicleType     | --- | Constant "BAKWAGEN"                                         |
| requestedVehicleCapacity | --- |  Constant 0                                       |
| plannedStartTime         | `<xs:element name='ArrivalTime' type='xs:dateTime' nillable='false' minOccurs='1' maxOccurs='1'/>` |                                         |
| plannedEndTime           | `<xs:element name='DepartureTime' type='xs:dateTime' nillable='false' minOccurs='1' maxOccurs='1'/>` | Among all stops within this trip, takes the latest (max) stop.departureTime  |
| plannedDepartureTime     | --- | Same value as plannedStartTime |
| superLorry               | --- | Constant: false |
| stops                    | `<xs:element name="Breaks" minOccurs='0' maxOccurs='1'>`, <br> `<xs:element name="Stops" minOccurs='1' maxOccurs='1'>`  |   |
| carrierUuid              | `<xs:element name='ResourceProviderID' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>`  | Looks up the carrier cache code from the SCT and returns its value. |
| plannedFleetNumber  | `<xs:element name='RouteID' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` |  No mapping Conversion |
| consignments | `<xs:element name="Stops" minOccurs='1' maxOccurs='1'>` | Extracting a list of distinct consignments from stops. |
| administrativeKm         | `<xs:element name='TotalPlannedKM' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>`  |  Get the first action of the first stop and take a nonEmpty TotalPlannedKm  |
| administrativeTime       | `<xs:element name='TotalPlannedDurationExclBreaks' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>` |  Get the action of the first Stop and extract nonEmpty value of TotalPlannedDurationExclBreaks. Time is formatted here: [def formatAdministrativeTime(administrativeTime: Double): Option[String]](https://github.com/simacan/planning-service/blob/f93ba524cd541dbf7e1ad1a53ac8efb340f3f877/services/quintiq/src/main/scala/com/simacan/quintiq_planning_service/planning_scp/quintiq_to_sct/parsing/AdministrativeFunctions.scala#L96) |
| customerData             | see Customer data  |   |
| carrierPlanning          | see carrierPlanning   |    |

## 1.4 Customer data

### 1.4.1 Customer data trip level
| SCT Trip field           | XML fields involved  | mapping description 
|--------------------------|----------------------|-----------------------------------------|
| tripType | `<xs:element name='TripType' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>`  | Get the first nonEmpty TripType field of an action. |
| weekNr | `<xs:element name='WeekNr' type='xs:integer' nillable='false' minOccurs='1' maxOccurs='1'/>` | No mapping conversion |
| routeId | `<xs:element name='RouteID' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | No mapping conversion |
| totalPlannedDurationInclBreaks | `<xs:element name='TotalPlannedDuration' type='xs:string' nillable='false' minOccurs='0' maxOccurs='1'/>` | Get the first nonEmpty field. This field might be different compared to "TotalPlannedDurationExclBreaks" since it may include break times. |
| MaximumCapacity1 | `<xs:element name='MaximumCapacity1' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | The MaximumCapacity1 is set for resourceTypeTrailer. Currently this value is not present in there. But we aggregate (sum) all the values in this field. This value reflects the capocity of a trailer in square meters. Capacity |
| temperatureGroupName | `<xs:element name='TemperatureGroupName' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | We aggregate all the TemperatureGroupName values into a set. |
| resourceTypeDriver | `<xs:element name='ResourceTypeGroup' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | If the value of ResourceTypeGroup is "Driver" then we take the value of the fields `ResourceID`, `ResourceType`, `ResourceProviderID`, `IsCharter`.|
| resourceTypeTruck | `<xs:element name='ResourceTypeGroup' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | Same as above but the ResourceTypeGroup can be [Truck, Truck_LZV, E_Truck, E_Truck_LZV, Rigid, Rigid_LZV, E_Rigid, E_Rigid_LZV]   |
| resourceTypeTrailer | `<xs:element name='ResourceTypeGroup' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | Same as above but the ResourceTypeGroup should be Trailer |
| groupingAddressLocationId | `<xs:element name='AddressID' type='xs:string' nillable='true' minOccurs='0' maxOccurs='1'/>` | Same as groupingAddressLocation but takes the locationId from the locationResponse  |
| resourceType | `<xs:element name='ResourceTypeGroup' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | This field is deprecated and should be phased out by 01.03.2025 ! |

### 1.4.2 Customer data stop level
| SCT Stop field           | XML fields involved  | mapping description 
|--------------------------|----------------------|-----------------------------------------|
| DistanceToNext | `<xs:element name='DistanceToNext' type='xs:double' nillable='false' minOccurs='1' maxOccurs='1'/>` | Distance to next stop. Value is extracted if it is higher than 0.  |


### 1.4.3 Customer data consignment level
| SCT Consignment field    | XML fields involved  | mapping description 
|--------------------------|----------------------|-----------------------------------------|
| packagingType | `<xs:element name='PackagingType' type='sc:nonEmptyString' nillable='false' minOccurs='1' maxOccurs='1'/>` | No mapping conversion |
| isCharter | `<xs:element name='IsCharter' type='xs:boolean' nillable='true' minOccurs='1' maxOccurs='1'/>` | Copied from the action level. That means every consignment on that action has the same value. Note: This should be phased out in favor of the isCharter on the customer data trip level. |
| isFinalShipmentForPicking | `<xs:element name='IsFinalShipmentForPicking' type='xs:boolean' nillable='false' minOccurs='1' maxOccurs='1'/>` | Takes the first value of possibly multiple values from the `ExternalOrder` (loadcarrier?) level. |
| isReleasedForProduction | `<xs:element name='IsReleasedForProduction' type='xs:boolean' nillable='false' minOccurs='1' maxOccurs='1'/>` | Copied from the action level (all consignments on that action will have the same value). |
| FinalDestination_AddressID | `<xs:element name='FinalDestination' type='sc:location' nillable='false' minOccurs='1' maxOccurs='1'/>` | Takes the `AddressID` property on the location type. Extracted from the first `ExternalOrder`. |
| FinalDestination_AddressGroupID | `<xs:element name='FinalDestination' type='sc:location' nillable='false' minOccurs='1' maxOccurs='1'/>` | Takes the `AddressGroupID` property on the location type. Extracted from the first `ExternalOrder`. |

### 1.4.4 Customer data goods (loadcarrier) level
| SCT Loadcarrier field    | XML fields involved  | mapping description 
|--------------------------|----------------------|-----------------------------------------|
| quantity1 | `<xs:element name='Quantity1' type='xs:double' nillable='false' minOccurs='1' maxOccurs='1'/>` | For each `ExternalOrder` we extract the value. Otherwise no conversion. Quantity1 is a square meter measurement. |
| Quantity3 | `<xs:element name='Quantity3' type='xs:double' nillable='false' minOccurs='1' maxOccurs='1'/>` | For each `ExternalOrder`. Otherwise no conversion. Quantity three is a qubic meter measurement. |
| blockId | `<xs:element name='BlockID' type='xs:string' nillable='true' minOccurs='0' maxOccurs='1'/>` | For each `ExternalOrder`. Otherwise no conversion. blockId is the identifier for the loadcarrier. |

Notes:

- RoundTrip: If it starts at the DC and ends at the DC but can contain multiple trips. is a RoundTrip.
- Jumbo SC sends DC ids in the form of "DC23". For these, it is necessary that the code is filled in in the custom(er) location id field for that location in Masterdata.
-  

