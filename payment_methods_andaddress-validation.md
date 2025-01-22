# Work Plan for Payment Methods and Address Validation

## For Tolu(Backend)

### API for Retrieving Saved Payment Methods
- **Endpoint**: `GET /orders/payment-methods`
- **Purpose**:
  - Retrieve a list of saved payment methods associated with a customer's Stripe Customer ID from the database.
  - Ensure the response includes:
    - Card type (e.g., Visa, MasterCard).
    - Last four digits of the card.
    - Expiration date.
  - Limit the response to the three most recent payment methods.

### API for Processing Saved Payment Methods
- **Endpoint**: `POST /orders/process-payment-method`
- **Purpose**:
  - Process a payment using a given payment method ID linked to a customer's Stripe account.
  - Validate the payment method ID to ensure it belongs to the correct Stripe Customer ID.
  - Include fallback mechanisms to:
    - Handle expired or invalid payment methods.
    - Return appropriate error messages and suggest alternatives if a payment fails.

### Update Webhook for Payment Method Management
- **Enhancements**:
  - Modify the existing Stripe webhook to:
    - Check the total number of saved payment methods for a customer.
    - Automatically delete the oldest methods if the total exceeds three, retaining only the most recent three.

---

## For Chinenye(Frontend)

### Shipping Address Validation with USPS API
- **Integration**:
  - Incorporate the USPS Address Validation API (or similar) into the checkout workflow.
- **Functionality**:
  - Validate customer-entered shipping addresses before submission.
  - Provide real-time feedback:
    - Suggest corrections for minor address discrepancies.
    - Display clear error messages for invalid or unsupported addresses.

### Displaying Saved Payment Methods
- **UI Integration**:
  - Use the `GET /orders/payment-methods` API to fetch and display the customer’s saved payment methods.
  - Display details such as:
    - Card type (e.g., Visa, MasterCard).
    - Last four digits of the card.
    - Expiration date.
  - Provide options for customers to:
    - Select an existing saved payment method.
    - Initiate a new payment method through a Stripe checkout session.

### Checkout with Saved Payment Methods
- **Implementation**:
  - Use the `POST /orders/process-payment-method` API to process payments with saved payment methods.
- **Error Handling**:
  - Manage cases like:
    - Expired or invalid payment methods.
    - Insufficient funds or declined payments.
  - Provide customers with options to:
    - Retry with another saved method.
    - Create a new payment method using a Stripe checkout session.

---

## For Tosin (Design)

### Design Validation Flow for Shipping Addresses
- **Goal**:
  - Create a user-friendly design to visually communicate the address validation process.
- **Flow**:
  - When a customer clicks "Submit," display a loading indicator or message indicating the address is being validated.
  - If the address is valid:
    - Show a confirmation message such as "Address validated successfully."
    - Automatically proceed with the order submission.
  - If the address is invalid:
    - Display an error message such as "Invalid address entered."
    - Provide suggested addresses based on the validation API, including nearby or corrected versions.
- **Deliverables**:
  - Mockups or prototypes for:
    - Loading/validation state.
    - Success message with smooth transition to order submission.
    - Error message and suggested address UI.
  - Ensure the design integrates seamlessly with the frontend's real-time feedback functionality.
