<!-- TODO: explain it also accept Debit card -->
<!-- TODO: explain when the card is checked for balance/limit, after 3DS -->
# Core API Card Transaction Integration
<hr>
One of the payment method offered by Midtrans is Card transaction. By using this payment method, customers will have the option to pay using credit card (or online-transaction-capable debit card) that is within Visa, MasterCard, JCB, or Amex network. Midtrans will also send real time notification when the customer complete the payment.

![visa](./../../asset/image/coreapi/visa.svg ":size=80") <br>
![mastercard](./../../asset/image/coreapi/mastercard.svg ":size=80") <br>
![jcb](./../../asset/image/coreapi/jcb.svg ":size=80") <br>
![amex](./../../asset/image/coreapi/american_express.svg ":size=80") <br>

Basic integration process of Card Transaction (3DS) will be explained below.

?> Please make sure you have already done [creating your Midtrans Account](/en/midtrans-account/overview.md), before proceeding with this section.

## Integration Step Overview
1. Get Card Token, via Frontend
2. Send transaction data to API Charge, via Backend
3. Open 3DS Authentication Page, via Frontend
4. Handle After Payment

<details>
<summary><b>Sequence Diagram</b></summary>
<article>
The overall Card Transaction (3DS) end-to-end payment proccess can be illustrated in following sequence diagram:

![3ds sequence diagram](./../../asset/image/core_api-sequence_3ds.png)
</article>
</details>

## 1. Get Card Token
Card `token_id` is representation of customer's card data, that will be used during a transaction. `token_id` should be retrieved using [MidtransNew3ds JS library](https://api.midtrans.com/v2/assets/js/midtrans-new-3ds.min.js) on merchant's website frontend, card data will be securely transmitted by frontend javascript to Midtrans API in exchange of card `token_id`, to avoid risk involved if card data being transmitted to merchant's backend.

### Include Midtrans JS
Include Midtrans JS library to our payment page, by adding this script tag:

```html
<script id="midtrans-script" type="text/javascript"
src="https://api.midtrans.com/v2/assets/js/midtrans-new-3ds.min.js" 
data-environment="sandbox" 
data-client-key="<INSERT YOUR CLIENT KEY HERE>"></script>
```

**Important**: Change the following attributes.

| Attribute | Value |
|-----------|-------|
| `data-environment`| Input `sandbox` or `production` (API environment)|
| `data-client-key`| Input **client key** [by following previous section.](/en/midtrans-account/overview.md#retrieving-api-access-keys) |

Link: [*More detailed definition*](https://api-docs.midtrans.com/#get-token)

### Get Card Token JS Implementation
To retrieve card `token_id`, we will be using `MidtransNew3ds.getCardToken` function. Implement the following Javascript on our payment page.

```javascript
// card data from customer input, for example
var cardData = {
  "card_number": 4811111111111114,
  "card_exp_month": 02,
  "card_exp_year": 2025,
  "card_cvv": 123,
};

// callback functions
var options = {
  onSuccess: function(response){
    // Success to get card token_id, implement as you wish here
    console.log('Success to get card token_id, response:', response);
    var token_id = response.token_id;

    console.log('This is the card token_id:', token_id);
    // Implement sending the token_id to backend to proceed to next step
  },
  onFailure: function(response){
    // Fail to get card token_id, implement as you wish here
    console.log('Fail to get card token_id, response:', response);

    // you may want to implement displaying failure message to customer.
    // Also record the error message to your log, so you can review
    // what causing failure for this transaction.
  }
};

// trigger `getCardToken` function
MidtransNew3ds.getCardToken(cardData, options);
```

You can use one of our test credentials for Card Payment:

Name | Value
--- | ---
Card Number | `4811 1111 1111 1114`
CVV | `123`
Exp Month | Any month (e.g: `02`)
Exp Year | Any future year (e.g: `2025`)
OTP/3DS | `112233`

Link: [*More testing credentials*](/en/technical-reference/sandbox-test.md).

### Get Card Token Response
If all goes well, we will be able to get card `token_id` from `response` object inside `onSuccess` callback function. It will be used as one of JSON parameter for [`/charge` API request](en/core-api/credit-card.md?id=charge-api-request).

`token_id` will need to be passed from frontend to backend for next step, it can be done using AJAX via Javascript, or html form POST, etc. Merchant are free to implement.

> **Note:** This `token_id` is only valid for 1 transaction. For each card transaction it is required to go through this process, to help ensure card data is transmitted securely. If you are looking to persist/save card token, you may use [One-click](https://api-docs.midtrans.com/#card-features-one-click)/[Two-clicks](https://api-docs.midtrans.com/#card-features-two-clicks) feature.

<details>
<summary><b>Sample Get Token Response</b></summary>
<article>

<!-- tabs:start -->
#### **Success Response**
Sample onSuccess `response` object:
```json
{
  "status_code": "200",
  "status_message": "Credit card token is created as Token ID.",
  "token_id": "481111-1114-77328ff4-eba6-4201-b31a-1070d8f19ae9",
  "hash": "481111-1114-xxxx"
}
```

#### **Failure Response**
Sample onFailure `response` object, it may contains the `validation_messages`:
```json
{
  "status_code": "400",
  "status_message": "One or more parameters in the payload is invalid.",
  "validation_messages": [
    "This card is not supported for online transactions. Please contact your bank", 
    "card_number does not match with luhn algorithm"
  ],
  "id": "02197189-7cab-4006-8379-51edcd0a253b"
}
```

<!-- tabs:end -->

</article>
</details>

## 2. Send Transaction Data to API Charge

API request should be done from **Merchant’s backend** to acquire `redirect_url` which will need to proceed to next step, opening 3DS authentication page by providing payment information. There are several components that are required:

Requirement | Description
--- | ---
Server Key | Explained on [previous section](/en/midtrans-account/overview.md)
`order_id` | Transaction order ID, defined from your side
`gross_amount` | Total amount of transaction, defined from your side
`token_id` | Represents customer's card information acquired from [Get Card Token Response](en/core-api/credit-card.md#get-card-token-response)
`authentication` | Flag to enable the 3D secure authentication.

?> **Note**: For better security & fraud prevention, you should set `authentication` to `true`. Only set `false` if you have confirmed with Midtrans & acquiring bank

### Charge API request

The example below shows a sample codes of the charge request:
<!-- tabs:start -->
#### **API-Request**

*This is an example in Curl, please implement according to your backend language, you can switch to other language on the "tab" above. (you can also check our [available language libraries](/en/technical-reference/library-plugin.md))*

#### Request Details
Type | Value
--- | ---
HTTP Method | `POST`
API endpoint (Sandbox) | `https://api.sandbox.midtrans.com/v2/charge`
API endpoint (Production) | `https://api.midtrans.com/v2/charge`

#### HTTP Headers
```
Accept: application/json
Content-Type: application/json
Authorization: Basic AUTH_STRING
```

**AUTH_STRING**: Base64(`ServerKey + :`)

?> Core API validates HTTP request by using Basic Authentication method. The username is your Server Key while the password is empty. The authorization header value is represented by AUTH_STRING. AUTH_STRING is base-64 encoded string of your username & password separated by **:** (colon symbol).

#### Full HTTP Request

```bash
curl -X POST \
  https://api.sandbox.midtrans.com/v2/charge \
  -H 'Accept: application/json'\
  -H 'Authorization: Basic <YOUR SERVER KEY ENCODED in Base64>' \
  -H 'Content-Type: application/json' \
  -d '{
  	"payment_type": "credit_card",
  	"transaction_details": {
    	"order_id": "order102",
    	"gross_amount": 789000
  	},
  	"credit_card": {
    	"token_id": "<token_id from Get Card Token Step>",
    	"authentication": true,
  	}
    "customer_details": {
        "first_name": "budi",
        "last_name": "pratama",
        "email": "budi.pra@example.com",
        "phone": "08111222333"
    }
}'
```

#### **PHP**

Install [**midtrans-php**](https://github.com/Midtrans/midtrans-php) library
```bash
composer require midtrans/midtrans-php
```

> Alternatively, if you are not using **Composer**, you can [download midtrans-php library](https://github.com/Midtrans/midtrans-php/archive/master.zip), and then require the file manually
> ```php
> require_once dirname(__FILE__) . '/pathofproject/Midtrans.php';
> ```

Card Transaction Charge
```php
// Set your Merchant Server Key
\Midtrans\Config::$serverKey = 'YOUR_SERVER_KEY';

$params = array(
    'transaction_details' => array(
        'order_id' => rand(),
        'gross_amount' => 10000,
    ),
	'payment_type' => 'credit_card',
    'credit_card'  => array(
        'token_id'      => $_POST['token_id'],
        'authentication'=> true,
    ),
    'customer_details' => array(
        'first_name' => 'budi',
        'last_name' => 'pratama',
        'email' => 'budi.pra@example.com',
        'phone' => '08111222333',
    ),
);

$response = \Midtrans\CoreApi::charge($params);
```

#### **Node JS**

Install [**midtrans-client**](https://github.com/Midtrans/midtrans-nodejs-client) NPM package
```bash
npm install --save midtrans-client
```

Card Transaction Charge
```javascript
const midtransClient = require('midtrans-client');
// Create Core API instance
let core = new midtransClient.CoreApi({
        isProduction : false,
        serverKey : 'YOUR_SERVER_KEY',
        clientKey : 'YOUR_CLIENT_KEY'
    });

let parameter = {
    "payment_type": "credit_card",
    "transaction_details": {
        "gross_amount": 12145,
        "order_id": "test-transaction-54321",
    },
    "credit_card":{
        "token_id": 'CREDIT_CARD_TOKEN', // change with your card token
        "authentication": true
    }
};

// charge transaction
core.charge(parameter)
    .then((chargeResponse)=>{
        console.log('chargeResponse:');
        console.log(chargeResponse);
    });
```

#### **Java**

Install [**midtrans-java**](https://github.com/Midtrans/midtrans-java) library

If you're using Maven as the build tools for your project, please add jcenter repository to your build definition, then add the following dependency to your project's build definition (pom.xml).
Maven:
```xml
<repositories>
    <repository>
        <id>jcenter</id>
        <name>bintray</name>
        <url>http://jcenter.bintray.com</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
      <groupId>com.midtrans</groupId>
      <artifactId>java-library</artifactId>
      <version>2.1.1</version>
    </dependency>
</dependencies>
```
Gradle:
If you're using Gradle as the build tools for your project, please add jcenter repository to your build script then add the following dependency to your project's build definition (build.gradle):
```bash
repositories {
    maven {
        url  "http://jcenter.bintray.com" 
    }
}

dependencies {
    compile 'com.midtrans:java-library:2.1.1'
}
```

Card Transaction Charge
```java
import com.midtrans.Config;
import com.midtrans.ConfigFactory;
import com.midtrans.service.MidtransCoreApi;
import com.midtrans.httpclient.error.MidtransError;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import org.json.JSONObject;


public class MidtransExample {

    public static void main(String[] args) throws MidtransError {
        MidtransCoreApi coreApi = new ConfigFactory(new Config("YOU_SERVER_KEY","YOUR_CLIENT_KEY", false)).getCoreApi();

		// Create Function JSON Raw Object
		public Map<String, Object> requestBody() {
		    UUID idRand = UUID.randomUUID();
		    Map<String, Object> params = new HashMap<>();
		    
		    Map<String, String> transactionDetails = new HashMap<>();
		    transactionDetails.put("order_id", idRand);
		    transactionDetails.put("gross_amount", "265000");
		    
		    Map<String, String> creditCard = new HashMap<>();
		    creditCard.put("token_id", YOUR_TOKEN_ID);
		    creditCard.put("authentication", "true");
		    
		    params.put("transaction_details", transactionDetails);
		    params.put("credit_card", creditCard);
		    
		    return params;
		}

		// charge transaction
		JSONObject result = coreApi.chargeTransaction(requestBody());
		System.out.println(result);
    }
}
```
#### **Python**

Install [**midtransclient**](https://github.com/Midtrans/midtrans-python-client) PIP package
```bash
pip install midtransclient
```

Card Transaction Charge
```python
import midtransclient
# Create Core API instance
core_api = midtransclient.CoreApi(
    is_production=False,
    server_key='YOUR_SERVER_KEY',
    client_key='YOUR_CLIENT_KEY'
)
# Build API parameter
param = {
    "payment_type": "credit_card",
    "transaction_details": {
        "gross_amount": 12145,
        "order_id": "test-transaction-54321",
    },
    "credit_card":{
        "token_id": 'CREDIT_CARD_TOKEN', # change with your card token
        "authentication": True
    }
}

# charge transaction
charge_response = core_api.charge(param)
```
<!-- tabs:end -->

?> **Optional:** You can customize [transaction_details](https://snap-docs.midtrans.com/#json-objects) data. To include data like `customer_details`, `item_details`, etc. It's recommended to send as much detail so on report/dashboard those information will be included.

### Charge API response
Upon successful request, you will get the **API response** like the following:

```json
{
  "status_code": "201",
  "status_message": "Success, Credit Card transaction is successful",
  "transaction_id": "0bb563a9-ebea-41f7-ae9f-d99ec5f9700a",
  "order_id": "order102",
  "redirect_url": "https://api.sandbox.veritrans.co.id/v2/token/rba/redirect/481111-1114-0bb563a9-ebea-41f7-ae9f-d99ec5f9700a",
  "gross_amount": "789000.00",
  "currency": "IDR",
  "payment_type": "credit_card",
  "transaction_time": "2019-08-27 15:50:54",
  "transaction_status": "pending",
  "fraud_status": "accept",
  "masked_card": "481111-1114",
  "bank": "bni",
  "card_type": "credit"
}
```

- If the `transaction_status` is `capture` and `fraud_status` is `accept`, it means the transaction require non 3DS, and is successfuly complete.

- If the `transaction_status` is `pending` and `redirect_url` exists, it means the transaction require 3DS, and we will need to proceed to next step, opening 3DS authentication page.

### Other Sample Response

Status Code | Description | Example
--- | --- | ---
200 | Success transaction complete (non 3DS transaction) | "transaction_status": "capture"
201 | Need to open the redirect_url (3DS transaction) | "https://api.sandbox.veritrans.co.id/v2/token/rba/redirect/481111-1114-f424a955-ed0f-4a64-88ea-60cdc9655984 "
401 | Failed. Wrong authorization sent  | "Access denied, please check client or server key"
4xx | Failed. Wrong parameter sent. Follow the error_message and check your parameter | "transaction_details.gross_amount is not equal to the sum of item_details"
5xx | Failed. Midtrans internal error. Most of the time this is temprorary, you can retry the request later | "Sorry, we encountered internal server error. We will fix this soon."

## 3. Open 3DS Authentication Page

As part of API response, we now have `redirect_url`. It should be opened (displayed to customer) using [MidtransNew3ds JS library](https://api.midtrans.com/v2/assets/js/midtrans-new-3ds.min.js) on merchant's website frontend.

To open 3DS page we can use `MidtransNew3ds.authenticate` or `MidtransNew3ds.redirect` function. Input the `redirect_url` retrieved previously.

### Open 3DS Authenticate Page JS Implementation

```javascript
var redirect_url = '<redirect_url Retrieved from Charge Response>';

// callback functions
var options = {
  performAuthentication: function(redirect_url){
    // Implement how you will open iframe to display 3ds authentication redirect_url to customer
    popupModal.openPopup(redirect_url);
  },
  onSuccess: function(response){
    // 3ds authentication success, implement payment success scenario
    console.log('response:',response);
    popupModal.closePopup();
  },
  onFailure: function(response){
    // 3ds authentication failure, implement payment failure scenario
    console.log('response:',response);
    popupModal.closePopup();
  },
  onPending: function(response){
    // transaction is pending, transaction result will be notified later via POST notification, implement as you wish here
    console.log('response:',response);
    popupModal.closePopup();
  }
};

// trigger `authenticate` function
MidtransNew3ds.authenticate(redirect_url, options);



/**
 * Example helper functions to open Iframe popup, you may replace this with your own method to open iframe
 * PicoModal library is used:
 * <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/picomodal/3.0.0/picoModal.js"></script>
 */
var popupModal = (function(){
  var modal = null;
  return {
    openPopup(url){
      modal = picoModal({
        content:'<iframe frameborder="0" style="height:90vh; width:100%;" src="'+url+'"></iframe>',
        width: "75%", 
        closeButton: false, 
        overlayClose: false,
        escCloses: false
      }).show();
    },
    closePopup(){
      try{
        modal.close();
      } catch(e) {}
    }
  }
}());

/**
 * Alternatively instead of opening 3ds authentication redirect_url using iframe,
 * you can also redirect customer using: 
 * MidtransNew3ds.redirect(redirect_url, { callbackUrl : 'https://mywebsite.com/finish_3ds' });
 **/
```

### 3DS Authenticate JSON Response
On the JS callback function, we will get the transaction result as JSON response like the followings.

<!-- tabs:start -->
#### **Success Response**
Sample of success transaction callback response:
```json
{
  "status_code": "200",
  "status_message": "Success, Credit Card transaction is successful",
  "channel_response_code": "00",
  "channel_response_message": "Approved",
  "bank": "bni",
  "eci": "05",
  "transaction_id": "405d27d5-5ad9-43ac-bdd6-0ccbde7d7dda",
  "order_id": "test-transaction-54321",
  "merchant_id": "G490526303",
  "gross_amount": "100000.00",
  "currency": "IDR",
  "payment_type": "credit_card",
  "transaction_time": "2020-08-12 16:04:23",
  "transaction_status": "capture",
  "fraud_status": "accept",
  "approval_code": "1597223068747",
  "masked_card": "481111-1114",
  "card_type": "credit"
}
```

#### **Failure Response**
Sample of failure transaction callback response:
```json
{
  "status_code": "202",
  "status_message": "Card is not authenticated.",
  "bank": "bni",
  "eci": "07",
  "transaction_id": "1063cc1f-f07e-4755-ab85-19a4592de097",
  "order_id": "test-transaction-54321",
  "merchant_id": "G490526303",
  "gross_amount": "100000.00",
  "currency": "IDR",
  "payment_type": "credit_card",
  "transaction_time": "2020-08-12 16:03:49",
  "transaction_status": "deny",
  "fraud_status": "accept",
  "masked_card": "481111-1114",
}
```
<!-- tabs:end -->

If the `transaction_status` is `capture` and `fraud_status` is `accept`, it means the transaction is success, and is now complete.

> **IMPORTANT NOTE:** To update transaction status on your backend/database, DO NOT solely rely on frontend callbacks! For security reason to make sure the status is authentically coming from Midtrans, only update transaction status based on HTTP Notification or [API Get Status](https://api-docs.midtrans.com/#get-transaction-status).

## 4. Handle After Payment

Other than customer being redirected, when the status of payment is updated/changed (i.e: payment has been successfully received), Midtrans will send **HTTP Notification** (or webhook) to your server's `Notification Url` (specified on Midtrans Dashboard, under menu **Settings > Configuration `Notification URL`**). Follow this link for more details:

<div class="my-card">

#### [Handle Webhook HTTP Notification](/en/after-payment/http-notification.md)
</div>

## Description

`transaction_status` value description for card transaction:

| Transaction Status | Description |
| ------------------ | ----------- |
| `capture` | Transaction successful, fund has been deducted |
| `pending` | Transaction is initiated and waiting for further action (3DS by customer) |
| `deny` | Transaction is denied, further check `channel_response_message` or `fraud_status` |
| `expire` | Transaction failure because customer did not complete 3DS within allowed time |

Link: [*More detailed definition of transaction_status & fraud_status*](/en/after-payment/status-cycle.md)

## Next Step:
<br>

<div class="my-card">

#### [Taking Action of Payment](/en/after-payment/overview.md)
</div>

<div class="my-card">

#### [Core API Advanced Feature](/en/core-api/advanced-features.md)
</div>

<div class="my-card">

#### [Transaction Status Cycle and Action](/en/after-payment/status-cycle.md)
</div>

<hr>

#### Reference:

> You can also refer to this sample implementation:
>	- [NodeJs - Express](https://github.com/Midtrans/midtrans-nodejs-client/blob/master/examples/expressApp/views/simple_core_api_checkout.ejs)
>	- [Python - Flask](https://github.com/Midtrans/midtrans-python-client/blob/master/examples/flask_app/templates/simple_core_api_checkout.html)

For more detail: [Complete Core API documentation](https://api-docs.midtrans.com/)