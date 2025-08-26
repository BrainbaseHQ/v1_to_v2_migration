# Brainbase V1 to Brainbase V2 Migration

## Summary

Version 1 of Brainbase is a deprecated software platform which is no longer actively maintained (besides existing infrastructure). Brainbase now offers Version 2, using a language/framework called Based. Based allows for far more deterministic flows. 

Version 1 currently supports ~100 active car dealerships. The goal is to migrate these (and future deployments) to Version 2.  

We currently support 3 integrations: XTime, Techwall, MyKaarma. We will start with Techwall as this represents the majority of our dealerships, and doesn't use expiring bearer tokens (making our lives a little easier)

We use a 'wrapper' API that sits on top of Techwalls API for a majority of these actions. The "documentation" for the wrapper can be found in **existing_dealership/tools**. Techwall documentation can be found in /**documentation**/*.

## Core Functionality
Below are the core “actions” the AI needs to take, with detailed notes on each. 

### Lookup Customer

At the beginning of each flow, you will have to lookup customer information given the callers phone number.

You recieve the customers phone number in Based like: ` phone_number = state.get('Caller', "+14156667777")`. You can directly call the Techwall API to get this data. 

``` 
Response Example:
{
	"success": true,
	"message": "",
	"customer": {
		"firstName": "GRATIS",
		"lastName": "TESTER",
		"email": "brenan@gratistech.com",
		"phone": "4238340853",
		"vehicles": [
			{
				"vin": "1FTFX1EF0DKD26366",
				"year": "2013",
				"make": "FORD",
				"model": "F-150",
				"mileage": "12348",
				"declinedServices": [
					{
						"originalDate": "2022-12-12",
						"serviceName": "Flush Transmission Fluid",
						"opCode": "55",
						"comments": "..."
					}
				],
				"recalls": [
					{
						"campaignNumber": "19V182000 ",
						"actionNumber": "EA15001 ",
						"reportDate": "2019-06-03",
						"component": "AIR BAGS:FRONTAL:DRIVER SIDE:INFLATOR MODULE ",
						"summary": "Honda (American Honda Motor Co.) is recalling specific 2003 Acura 3.2CL, 2013-2016 ILX ...",
						"consequence": "An explosion of an inflator within the driver frontal air bag module may result in ...",
						"remedy": "Honda will notify owners, and dealers will replace the driver's air bag ...",
						"notes": "Owners may also contact the ..."
					}
				]
			}
		],
		"appointments": [
			{
				"id": "54022",
				"appointment_date_time": "2023-09-21 10:00:00",
				"vehicle_year": "2013",
				"vehicle_model": "F-150",
				"vehicle_make": "FORD",
				"created_on": "2023-09-08 06:44:17",
				"advisorName": "Dalton Harris",
				"advisorId": "278925",
				"transportType": "Drop-Off",
				"services": [
					{
						"serviceName": "OIL AND FILTER CHANGE",
						"opCode": "00",
						"comments": ""
					},
					{
						"serviceName": "CUSTOMER STATES PERFORM CAMPAIGN 172",
						"opCode": "172",
						"comments": ""
					}
				]
			}
		]
	}
}

```

### Create Customer*
This is handled for you in the wrapper whenever you go to book an appointment for a customer that doesn't exist. If the customer doesn't have an account, you just need to ask them for: 
* First Name
* Last Name
* Vehicle Make
* Vehicle Model
* Vehicle Year
* Email Address

and pass these into the API call for booking an appointment

### Get Availability
Whenever a customer wants to book or reschedule an appointment, you must lookup availability for the customer. This is the payload required (as an OpenAPI spec): 

```
      "parameters": {
        "type": "object",
        "properties": {
          "startDate": {
            "type": "string",
            "description": "A datetime string that specifies the start date and time of the appointment range."
          },
          "endDate": {
            "type": "string",
            "description": "A datetime string that specifies the end date and time of the appointment range."
          },
          "vehicle": {
            "type": "object",
            "properties": {
              "year": {
                "type": "string",
                "description": "The year of the vehicle."
              },
              "make": {
                "type": "string",
                "description": "The make of the vehicle."
              },
              "model": {
                "type": "string",
                "description": "The model of the vehicle."
              }
            },
            "required": [
              "year",
              "make",
              "model"
            ]
          },
          "transportationType": {
            "type": "string",
            "description": "The type of transportation the user is looking for. Only include one of the options",
            "enum": [
              "PICKUP",
              "WAITER",
              "DROPOFF"
            ]
          },
          "endpoint": {
            "type": "string",
            "description": "this will always be /dgaauto/techwall/getavailability/randy-marion-buick-gmc",
            "enum": [
              "/dgaauto/techwall/getavailability/randy-marion-buick-gmc"
            ]
          },
          "dealerId": {
            "type": "string",
            "description": "The ID of the dealership.",
            "enum": [
              "1"
            ]
          },
          "opCodes": {
            "type": "string",
            "description": "Comma-delimited list of opCodes for the appointment as a string. OpCodes are the codes associated with the appointment type. For example, if the user is looking for an OIL AND FILTER CHANGE the opCode would be 01 HVP2. Reference your system prompts for the correct opCode mappings."
          }
        },
        "required": [
          "startDate",
          "endDate",
          "dealerId"
        ]
      }

```

This is an example response: 

```
{'statusCode': 200, 'body': '{"open_available_appointment_slots": "Tuesday, August 26th - 7:00 AM, Tuesday, August 26th - 7:15 AM, Tuesday, August 26th - 7:30 AM, Tuesday, August 26th - 7:45 AM, Tuesday, August 26th - 8:00 AM, Tuesday, August 26th - 8:15 AM, Tuesday, August 26th - 8:30 AM, Tuesday, August 26th - 8:45 AM, Tuesday, August 26th - 9:00 AM, Tuesday, August 26th - 9:15 AM, Tuesday, August 26th - 9:30 AM, Tuesday, August 26th - 9:45 AM, Tuesday, August 26th - 10:00 AM, Tuesday, August 26th - 10:15 AM, Tuesday, August 26th - 10:30 AM, Tuesday, August 26th - 10:45 AM, Tuesday, August 26th - 11:00 AM, Tuesday, August 26th - 11:15 AM, Tuesday, August 26th - 11:30 AM, Tuesday, August 26th - 11:45 AM, Tuesday, August 26th - 12:00 PM, Tuesday, August 26th - 12:15 PM, Tuesday, August 26th - 12:30 PM, Tuesday, August 26th - 12:45 PM, Tuesday, August 26th - 1:00 PM, Tuesday, August 26th - 1:15 PM, Tuesday, August 26th - 1:30 PM, Tuesday, August 26th - 1:45 PM, Tuesday, August 26th - 2:00 PM, Tuesday, August 26th - 2:15 PM, Tuesday, August 26th - 2:30 PM, Tuesday, August 26th - 2:45 PM, Tuesday, August 26th - 3:00 PM, Tuesday, August 26th - 3:15 PM, Tuesday, August 26th - 3:30 PM, Tuesday, August 26th - 3:45 PM, Tuesday, August 26th - 4:00 PM, Tuesday, August 26th - 4:15 PM, Tuesday, August 26th - 4:30 PM, Tuesday, August 26th - 4:45 PM, Tuesday, August 26th - 5:00 PM, Tuesday, August 26th - 5:15 PM, Tuesday, August 26th - 5:30 PM, Tuesday, August 26th - 5:45 PM, Wednesday, August 27th - 7:00 AM, Wednesday, August 27th - 7:15 AM, Wednesday, August 27th - 7:30 AM, Wednesday, August 27th - 7:45 AM, Wednesday, August 27th - 8:00 AM, Wednesday, August 27th - 8:15 AM, Wednesday, August 27th - 8:30 AM, Wednesday, August 27th - 8:45 AM, Wednesday, August 27th - 9:00 AM, Wednesday, August 27th - 9:15 AM, Wednesday, August 27th - 9:30 AM, Wednesday, August 27th - 9:45 AM, Wednesday, August 27th - 10:00 AM, Wednesday, August 27th - 10:15 AM, Wednesday, August 27th - 10:30 AM, Wednesday, August 27th - 10:45 AM, Wednesday, August 27th - 11:00 AM, Wednesday, August 27th - 11:15 AM, Wednesday, August 27th - 11:30 AM, Wednesday, August 27th - 11:45 AM, Wednesday, August 27th - 12:00 PM, Wednesday, August 27th - 12:15 PM, Wednesday, August 27th - 12:30 PM, Wednesday, August 27th - 12:45 PM, Wednesday, August 27th - 1:00 PM, Wednesday, August 27th - 1:15 PM, Wednesday, August 27th - 1:30 PM, Wednesday, August 27th - 1:45 PM, Wednesday, August 27th - 2:00 PM, Wednesday, August 27th - 2:15 PM, Wednesday, August 27th - 2:30 PM, Wednesday, August 27th - 2:45 PM, Wednesday, August 27th - 3:00 PM, Wednesday, August 27th - 3:15 PM, Wednesday, August 27th - 3:30 PM, Wednesday, August 27th - 3:45 PM, Wednesday, August 27th - 4:00 PM, Wednesday, August 27th - 4:15 PM, Wednesday, August 27th - 4:30 PM, Wednesday, August 27th - 4:45 PM, Wednesday, August 27th - 5:00 PM, Wednesday, August 27th - 5:15 PM, Wednesday, August 27th - 5:30 PM, Wednesday, August 27th - 5:45 PM, Thursday, August 28th - 7:00 AM, Thursday, August 28th - 7:15 AM, Thursday, August 28th - 7:30 AM, Thursday, August 28th - 7:45 AM, Thursday, August 28th - 8:00 AM, Thursday, August 28th - 8:15 AM, Thursday, August 28th - 8:30 AM, Thursday, August 28th - 8:45 AM, Thursday, August 28th - 9:00 AM, Thursday, August 28th - 9:15 AM, Thursday, August 28th - 9:30 AM, Thursday, August 28th - 9:45 AM, Thursday, August 28th - 10:00 AM, Thursday, August 28th - 10:15 AM, Thursday, August 28th - 10:30 AM, Thursday, August 28th - 10:45 AM, Thursday, August 28th - 11:00 AM, Thursday, August 28th - 11:15 AM, Thursday, August 28th - 11:30 AM, Thursday, August 28th - 11:45 AM, Thursday, August 28th - 12:00 PM, Thursday, August 28th - 12:15 PM, Thursday, August 28th - 12:30 PM, Thursday, August 28th - 12:45 PM, Thursday, August 28th - 1:00 PM, Thursday, August 28th - 1:15 PM, Thursday, August 28th - 1:30 PM, Thursday, August 28th - 1:45 PM, Thursday, August 28th - 2:00 PM, Thursday, August 28th - 2:15 PM, Thursday, August 28th - 2:30 PM, Thursday, August 28th - 2:45 PM, Thursday, August 28th - 3:00 PM, Thursday, August 28th - 3:15 PM, Thursday, August 28th - 3:30 PM, Thursday, August 28th - 3:45 PM, Thursday, August 28th - 4:00 PM, Thursday, August 28th - 4:15 PM, Thursday, August 28th - 4:30 PM, Thursday, August 28th - 4:45 PM, Thursday, August 28th - 5:00 PM, Thursday, August 28th - 5:15 PM, Thursday, August 28th - 5:30 PM, Thursday, August 28th - 5:45 PM"}'}


```
We do some additional handling on the wrapper to account for vehicle names and other edge cases that can cause issues.

### Book Appointment

You can only do this after getting availability. You will also need the AI to have context to something called OpCodes (which dictate which services are required). You will find an example of these in existing_dealership/prompt.txt. You can also access them using the TechWall API.

The payload looks like this: 

```
      "parameters": {
        "type": "object",
        "properties": {
          "opCodes": {
            "type": "string",
            "description": "Comma-delimited list of opCodes for the appointment as a string. OpCodes are the codes associated with the appointment type. For example, if the user is looking for an OIL AND FILTER CHANGE the opCode would be 01 HVP2. Reference your system prompts for the correct opCode mappings."
          },
          "transportationType": {
            "type": "string",
            "description": "The type of transportation the user is looking for. Only include one of the options",
            "enum": [
              "PICKUP",
              "WAITER",
              "DROPOFF"
            ]
          },
          "endpoint": {
            "type": "string",
            "description": "this will always be /dgaauto/techwall/bookappointment/randy-marion-buick-gmc",
            "enum": [
              "/dgaauto/techwall/bookappointment/randy-marion-buick-gmc"
            ]
          },
          "dateTime": {
            "type": "string",
            "description": "The date and time of the desired appointment in datetime string."
          },
          "dealerId": {
            "type": "string",
            "description": "The ID of the dealership.",
            "enum": [
              "1"
            ]
          },
          "firstName": {
            "type": "string",
            "description": "The first name of the caller."
          },
          "lastName": {
            "type": "string",
            "description": "The last name of the caller."
          },
          "comment": {
            "type": "string",
            "description": "Always precede the comment with 'CUSTOMER STATES:', and then a comment with the customer's service request and transportation request. This should be the speech to text raw response from the customer, or a summary as close to the original request as possible. Make an explanation of exactly what the customer needs ie. their breaks are squeeking. Also include the transportation type they would like."
          },
          "phoneNumber": {
            "type": "string",
            "description": "The phone number of the caller, most likely found in the additional information. DO NOT include the country code. for example, +14154650216 should be 4154650216"
          },
          "vehicle": {
            "type": "object",
            "properties": {
              "year": {
                "type": "string",
                "description": "The year of the vehicle."
              },
              "make": {
                "type": "string",
                "description": "The make of the vehicle."
              },
              "model": {
                "type": "string",
                "description": "The model of the vehicle."
              }
            },
            "required": [
              "year",
              "make",
              "model"
            ]
          }
        },
        "required": [
          "dealerId",
          "dateTime",
          "comment",
          "phoneNumber"
        ]
      }
```

### Cancel Appointment
For cancelling an appointment, you can use the Techwall API directly. This only requires the appointment ID, which can always be found in the customer details that come back. You just need to confirm with the customer that they want it to be cancelled.
### Reschedule Appointment
For rescheduling an appointment, you can again use the wrapper. The payload follows the following schema (all provided in /existing_dealership):

```
      "parameters": {
        "type": "object",
        "properties": {
          "opCodes": {
            "type": "string",
            "description": "Comma-delimited list of opCodes for the appointment as a string. Include ALL of the opcodes from the original appointment that they wish to reschedule."
          },
          "dealerId": {
            "type": "string",
            "description": "The ID of the dealership.",
            "enum": [
              "1"
            ]
          },
          "transportationType": {
            "type": "string",
            "description": "The type of transportation the user is looking for. Only include one of the options",
            "enum": [
              "PICKUP",
              "WAITER",
              "DROPOFF"
            ]
          },
          "dateTime": {
            "type": "string",
            "description": "The date and time of the new desired rescheduled appointment in datetime string."
          },
          "firstName": {
            "type": "string",
            "description": "The first name of the caller."
          },
          "endpoint": {
            "type": "string",
            "description": "this will always be /dgaauto/techwall/reschedule/randy-marion-buick-gmc",
            "enum": [
              "/dgaauto/techwall/reschedule/randy-marion-buick-gmc"
            ]
          },
          "lastName": {
            "type": "string",
            "description": "The last name of the caller."
          },
          "comment": {
            "type": "string",
            "description": "The same comment/summary that exists on the appointment they wish to reschedule."
          },
          "phoneNumber": {
            "type": "string",
            "description": "The phone number of the caller, most likely found in the additional information. DO NOT include the country code. For example, +14154650216 should be 4154650216."
          },
          "appointmentId": {
            "type": "string",
            "description": "The ID of the appointment to be rescheduled."
          },
          "vehicle": {
            "type": "object",
            "properties": {
              "year": {
                "type": "string",
                "description": "The year of the vehicle."
              },
              "make": {
                "type": "string",
                "description": "The make of the vehicle."
              },
              "model": {
                "type": "string",
                "description": "The model of the vehicle."
              }
            },
            "required": [
              "year",
              "make",
              "model"
            ]
          }
        },
        "required": [
          "opCodes",
          "dealerId",
          "transportationType",
          "dateTime",
          "firstName",
          "lastName",
          "comment",
          "phoneNumber",
          "appointmentId",
          "vehicle",
          "endpoint"
        ]
      }
```
## How to get started

Start by creating a relativley simple flow with all of the above functionality. The ordering and process is outlined in prompt.txt in detail.