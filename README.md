# Food Delivery Service implemented using Conductor
Sample Java Application demonstrating Saga microservice architecture pattern for a food delivery app.

See: https://medium.com/orkes/saga-pattern-in-distributed-systems-f11b9a2221f5 for the blog

## Running this Example

### Environment setup
1. Install JAVA 17 - https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html
2. Install sqlite - https://www.tutorialspoint.com/sqlite/sqlite_installation.htm.
   If using brew, you can just run ```brew install sqlite```

### Workflow Setup on Orkes Playground

#### Use existing out-of-the-box workflows
We have already setup these workflows with all the necessary permissions. These can be used directly, if you don't need to modify these workflows.
1. Food Delivery Workflow - https://play.orkes.io/workflowDef/FoodDeliveryWorkflow
2. Cancellation workflow - https://play.orkes.io/workflowDef/FoodDeliveryFailureWorkflow

### Running the application and workers locally

1. Clone the project to your local
2. Update following properties in [application.properties](src/main/resources/application.properties)   
   1. `conductor.security.client.key-id` and `conductor.security.client.secret` are **NOT** set, please set them
      * When connecting to playground - refer to this [article](https://orkes.io/content/how-to-videos/access-key-and-secret) to get a key and secret
      * When connecting locally - follow the instructions [here (Install and Run Locally)](https://orkes.io/content/get-orkes-conductor)
   2. `conductor.worker.all.domain` is set to 'saga' by default, please change it to <yourname> or something else to avoid conflicts with workflows and workers spun up by others
3. From the root project folder, run `gradlew bootRun`

### Order Creation

We can use two approaches:
1. Call the triggerRideBookingFlow API from within the Application
   ```
    curl --location 'http://localhost:8081/startFoodDeliveryWorkflow' \
    --header 'Content-Type: application/json' \
    --data '{
         "customerId": 1,
         "address": "350 East 62nd Street, NY 10065",
         "contact": "+1(605)123-5674",
         "restaurantId": 2,
         "foodItems": [
         {
            "item": "Chicken with Broccoli",
            "quantity": 1
         },
         {
            "item": "Veggie Fried Rice",
            "quantity": 1
         },
         {
            "item": "Egg Drop Soup",
            "quantity": 2
         }],
         "additionalNotes": [
           "Do not put spice.",
           "Send cutlery."
         ],
         "deliveryInstructions": "Leave at the door!",
         "paymentAmount": 45.34,
         "paymentMethod": {
         "type": "Credit Card",
         "details": {
           "number": "1234 4567 3325 1345",
           "expiry": "05/2025",
           "cvv": "123"
         }
       }
     }'
   ```
2. Directly call the Orkes API for creating a workflow
   1. Generate a JWT token by following steps mentioned [here](https://orkes.io/content/access-control-and-security/applications#generating-token)
   2. Make an HTTPS request from postman/curl similar to below after replacing \<JWT Token\>:
    ``` 
    curl --location 'https://play.orkes.io/api/workflow' \
    --header "Content-Type: application/json" \
    --header 'X-Authorization: <JWT Token>' \
    --request POST \
    --data \ 
    ' 
    {
       "name": "FoodDeliveryWorkflow",
       "version": 1,
       "input": {
         "customerId": 1,
         "address": "350 East 62nd Street, NY 10065",
         "contact": "+1(605)123-5674",
         "restaurantId": 2,
         "foodItems": [
         {
            "item": "Chicken with Broccoli",
            "quantity": 1
         },
         {
            "item": "Veggie Fried Rice",
            "quantity": 1
         },
         {
            "item": "Egg Drop Soup",
            "quantity": 2
         }],
         "additionalNotes": [
           "Do not put spice.",
           "Send cutlery."
         ],
         "deliveryInstructions": "Leave at the door!",
         "paymentAmount": 45.34,
         "paymentMethod": {
         "type": "Credit Card",
         "details": {
           "number": "1234 4567 3325 1345",
           "expiry": "05/2025",
           "cvv": "123"
         }
       }
     }
     "taskToDomain": {
       "*": "saga"
     }}
   }'
   ```
   
A successful order creation workflow run will look like this:

![Screenshot 2024-02-26 at 11 38 14 AM](https://github.com/conductor-sdk/conductor-examples-food-delivery/assets/127052609/377549ec-58b0-425c-922b-6318a20b68c8)


#### Triggering the cancellation workflow to simulate rollback of distributed transactions

* Create a booking for rider 3 who doesn't have a payment method seeded.
``` 
curl --location 'https://play.orkes.io/api/workflow' \
--header "Content-Type: application/json" \
--header 'X-Authorization: <JWT Token>' \
--request POST \
--data '{
  "name": "cab_service_saga_booking_wf",
  "version": 1,
  "input": {
    "name": "FoodDeliveryWorkflow",
    "version": 1,
    "input": {
      "customerId": 1,
      "address": "350 East 62nd Street, NY 10065",
      "contact": "+1(605)123-5674",
      "restaurantId": 2,
      "foodItems": [
        {
          "item": "Chicken with Broccoli",
          "quantity": 1
        },
        {
          "item": "Veggie Fried Rice",
          "quantity": 1
        },
        {
          "item": "Egg Drop Soup",
          "quantity": 2
        }
      ],
      "additionalNotes": [
        "Do not put spice.",
        "Send cutlery."
      ],
      "deliveryInstructions": "Leave at the door!",
      "paymentAmount": 45.34,
      "paymentMethod": {
        "type": "Credit Card",
        "details": {
          "number": "1234 4567 3325 1345",
          "expiry": "05/2023",
          "cvv": "123"
        }
      }
    }
  },
  "taskToDomain": {
    "*": "saga"
  }
}
'
```

* This will cause the workflow to fail and trigger the cancellation workflow.
* Failed workflow run will look like this:

    <img width="445" alt="Screenshot 2023-07-12 at 12 14 29 AM" src="https://github.com/conductor-sdk/conductor-examples-saga-pattern/assets/127052609/06c6ec04-1784-4916-ba63-0c03e5af43bc">


* A cancellation workflow run will look like this:

    <img width="450" alt="Screenshot 2023-07-12 at 12 15 30 AM" src="https://github.com/conductor-sdk/conductor-examples-saga-pattern/assets/127052609/37406ad6-f3e8-426d-898e-fe42768fc6d6">

* In the above workflow diagram, the simulated distributed rollback can be seen. The rollback sequence in case of failure occurring while payment processing is as follows:
  1. Shipment assignment is cancelled the Shipment Service
  2. Payment is cancelled in the Payment Service
  3. Order is cancelled in the Order Service
