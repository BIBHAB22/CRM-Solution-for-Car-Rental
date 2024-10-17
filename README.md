# **Car Rental CRM Project**

This project is a **Car Rental CRM** application developed using Salesforce to streamline and automate the car rental process. The application includes features for managing customers, car bookings, billing, and automated reminders for overdue payments, providing an efficient solution for car rental businesses.

## **Objectives**

The project aims to achieve the following business goals:
- Automate the car rental process to minimize manual work.
- Track and manage customer rental history, booking information, and billing.
- Provide reminders for overdue payments to ensure timely collection.
- Generate reports on car rentals, vehicle usage, and customer data for better decision-making.

### **Specific Outcomes**
- Automated billing calculations based on rental duration and payment status.
- Easy management of customer, car, and booking data.
- Implementation of reminders for overdue payments.
- Efficient tracking of payment statuses and rental fees.

## **Salesforce Key Features and Concepts Utilized**

- **Objects and Relationships**: Custom objects for **Customers**, **Bookings**, **Cars**, and **Billing**.
- **Automated Processes**: Record-Triggered Flows, Scheduled Apex for payment reminders, and Screen Flows for customer registration and car booking.
- **Formula Fields**: Used to calculate total rental days and amounts.
- **Validation Rules**: Ensuring data accuracy, such as valid rental periods and non-negative mileage.
- **Security**: Role-based access control for different users in the system.

## **Apex Code for Schedulable Class Payment Reminder**
```java
public class RemindUnpaidBookings implements Schedulable {
    
    // This method is called when the scheduled job runs
    public void execute(SchedulableContext sc) {
        // Calculate the date three days ago from today
        Date targetDate = Date.today().addDays(-3);
        
        // Query Booking__c records where:
        // - End_Date__c is exactly three days ago
        // - Payment_Status__c is not 'Paid'
        // - Customer_ID__r.Email__c is not null
        List<Car_Booking__c> unpaidBookings = [
            SELECT Id, 
                   Booking_ID__c, 
                   Customer_ID__r.Email__c, 
                   Customer_ID__r.Name, 
                   Payment_Status__c, 
                   End_Date__c
            FROM Car_Booking__c
            WHERE End_Date__c <= :targetDate
              AND Payment_Status__c != 'Paid'
              AND Customer_ID__r.Email__c != null
        ];
        
        // Prepare a list to hold email messages
        List<Messaging.SingleEmailMessage> emails = new List<Messaging.SingleEmailMessage>();
        
        // Iterate through each unpaid booking and compose an email
        for (Car_Booking__c booking : unpaidBookings) {
            Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
            
            // Set the recipient email address
            email.setToAddresses(new String[] { booking.Customer_ID__r.Email__c });
            
            // Set the email subject
            email.setSubject('Payment Reminder for Your Car Rental Booking #' + booking.Booking_ID__c);
            
            // Compose the email body
            String body = 'Dear ' + booking.Customer_ID__r.Name + ',\n\n' +
                          'We hope you enjoyed your recent car rental with us. Our records indicate that the payment for your booking #' + 
                          booking.Booking_ID__c + ' has not been received yet.\n\n' +
                          'Please make the payment at your earliest convenience to avoid any late fees or penalties.\n\n' +
                          'If you have already made the payment, please disregard this email or contact our support team for assistance.\n\n' +
                          'Thank you for choosing our services!\n\n' +
                          'Best regards,\n' +
                          'Your Car Rental Team';
            
            // Set the plain text body of the email
            email.setPlainTextBody(body);
            
            // Optionally, set the sender display name and email address
            email.setSenderDisplayName('Car Rental Team');
            // email.setReplyTo('support@yourcarrental.com'); // Uncomment and set if needed
            
            // Add the email to the list
            emails.add(email);
        }
        
        // Send all composed emails
        if (!emails.isEmpty()) {
            Messaging.sendEmail(emails);
        }
    }
}
```

## **Apex Trigger for Payment Status**
```java
trigger PaymentStatusTrigger on Billing__c (after update) {
    // List to store cars that need to be updated
    List<Car__c> carsToUpdate = new List<Car__c>();
    
    // List to store emails to be sent
    List<Messaging.SingleEmailMessage> emailsToSend = new List<Messaging.SingleEmailMessage>();
    
    // Iterate over the updated billing records
    for (Billing__c billing : Trigger.new) {
        // Check if the payment status is updated to 'Paid'
        if (billing.Payment_Status__c == 'Paid' && Trigger.oldMap.get(billing.Id).Payment_Status__c != 'Paid') {
            
            // Find the associated car using the Car__c field in Billing__c
            Car__c car = [SELECT Id, Status__c, Car_Model__c FROM Car__c WHERE Id = :billing.Car__c LIMIT 1];
            
            // Update the car status to 'Available'
            car.Status__c = 'Available';
            carsToUpdate.add(car);
            
            // Prepare the email message
            Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
            email.setToAddresses(new String[] {billing.Email__c}); // Assuming Customer_Email__c stores the user's email address
            email.setSubject('Payment Confirmation - Your Car Booking');
            email.setHtmlBody(
                'Dear ' + billing.Customer_Name__c + ',<br/><br/>' + // Assuming Customer_Name__c holds the customer's name
                'Thank you for your payment. Your payment for the car model ' + car.Car_Model__c + ' is now done.<br/>' +
                'We look forward to serving you.<br/><br/>' +
                'Best regards,<br/>' +
                'Car Rental Team'
            );
            emailsToSend.add(email);
        }
    }
    
    // Update car records if there are any to update
    if (!carsToUpdate.isEmpty()) {
        update carsToUpdate;
    }
    
    // Send confirmation emails if there are any to send
    if (!emailsToSend.isEmpty()) {
        Messaging.sendEmail(emailsToSend);
    }
}
```

## **Validation Rules**

### **Customer_Main__c Object:**
1. **Email Format Validator**: Ensures that the Email__c field contains a valid email format.
2. **Phone Number Validator**: Ensures that the Phone_Number__c contains only numeric values.


### **Booking__c Object**
1. **End Date Validator**: Ensures that the `End_Date__c` cannot be before the `Start_Date__c`.
2. **Payment Status Validator**: Ensures that the `Payment_Status__c` cannot be "Paid" unless a `Payment_Date__c` is provided.
3. **Booking Date Validator**: Ensures that the `Booking_Date__c` is not in the future.

### **Car__c Object**
1. **Mileage Validator**: Ensures that `Mileage__c` cannot be negative.
2. **Rental Price Validator**: Ensures that `Rental_Price_per_Day__c` is greater than zero.

### **Billing__c Object**
1. **Payment Date Validator**: Ensures that the `Payment_Date__c` cannot be before the `Booking_Date__c`.
2. **Total Amount Validator**: Ensures that `Total_Rental_Amount_Input_c__c` is greater than zero.
3. **Payment Status Validator**: Ensures that the `Payment_Status__c` cannot be "Paid" if `Payment_Date__c` is empty.

## **Testing and Validation**

The project has been thoroughly tested through:
- **Unit Testing**: Apex classes and triggers have been unit tested to ensure functionality.
- **User Interface Testing**: Manual and automated tests have been conducted on the UI to ensure smooth customer interactions, including the registration and booking flows.

## **Key Scenarios Addressed by Salesforce**

- Efficient handling of customer data through Lookup fields and custom objects.
- Automated billing calculation based on rental durations and payment statuses.
- Notifications sent automatically to customers when a payment is overdue.
- Streamlined process for renting cars, updating vehicle status, and returning cars, reducing administrative effort.

## **Documentation**

For a detailed description of the project, including objectives, key features, and testing, refer to the project documentation:
📝 [Project Documentation](https://docs.google.com/document/d/1F0exULvFkEXaO_ToIt6DtzpvKvlao0Ilu27NgbbxRzk/edit?usp=sharing)

## **Video Link**
For a detailed demonstration video click here: 🎥 [Video Demo](#)

---

Feel free to explore the project and contribute if you'd like to enhance its features or improve the functionality!

---
