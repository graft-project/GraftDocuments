# POS and Wallet CLI apps for RTA flow development/testing

## POS app
Suggested name: `rta-pos-cli`

### Workflow:
- once started, app initiates a test sale ([see p1 - p4 in this document](https://github.com/graft-project/DesignDocuments/blob/master/RFCs/%5BRFC-003-RTVF%5D-RTA-Transaction-Validation-Flow.md)) using [/presale](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#presale---supernode-returns-auth-sample-for-given-payment-id) and [/sale](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#sale---process-sale) interfaces. User should be able to setup a list of purchase items and transaction amount;  
- app mocks QR code generation by writing the information to be transferred via QR code to the local file (information could be encoded as json). File should be removed on app quit. Wallet app should mock QR code reading by reading from this file;  
- POS app starts polling for a payment status/state change according the flow using [/get_payment_status](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#getpaymentstatus---returns-payment-status-for-given-payment-id) interface. Every state change should be logged to stdout;  
- once POS app receives "pending" status for the sale it sends transaction request to a POS proxy supernode ([see p14](https://github.com/graft-project/DesignDocuments/blob/master/RFCs/%5BRFC-003-RTVF%5D-RTA-Transaction-Validation-Flow.md)) with `/get_tx` interface (TODO: currently missing, should be introduced);  
- once POS app receives encrypted transaction - POS validates  and signs tx if validation passed or broadcasts status "fail" for this tx otherwise ([see p16](https://github.com/graft-project/DesignDocuments/blob/master/RFCs/%5BRFC-003-RTVF%5D-RTA-Transaction-Validation-Flow.md))
- POS keeps polling it's proxy supernode for the payment status change - until it gets finite state (Completed or Failed) or timeout triggered. Once one of these events occurs - POS prints the message to stdout

### CLI interface
- _**qrcode-file**_ - filename where to write information to be transferred to a wallet file via QR-code. Default -value `wallet-qr-code.json`  
- _**amount**_ - sale amount value as a floating point number. Default value should be e.g. _10.123 GRFT_    
- _**sale-items**_ - filename where to read sale items from (json-encoded, but exact scheme to be defined). if not specified - some hardcoded sale items should be used
- _**sale-timeout**_ - defines a timeout for a sale. App starts counting time once status changed to `pending`;  
- _**supernode-address**_ - supernode address in `hostname:port` form. default is `localhost:28900`

## Wallet app
Suggested name: `rta-wallet-cli`

### Workflow:
- once started, app tries to open the wallet (using libwallet interfaces). Wallet name should be passed as a CLI option. If wallet can't be opened or libwallet can't connect to a daemon - all prints error to stderr and exits;
- app reads sale information prepared by POS app. If file can't be read or sale amount is bigger than available balance - quit with error message;  
- app requests for additional payment data it's proxy supernode ([see p5](https://github.com/graft-project/DesignDocuments/blob/master/RFCs/%5BRFC-003-RTVF%5D-RTA-Transaction-Validation-Flow.md)) using [/get_payment_data](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#getpaymentdata---returns-payment-data-for-given-payment-id-and-block-number-and-block-hash) interface. In case error (no connection to proxy supernode) - app prints message to stderr and quits. In case wallet proxy supernode returned HTTP 202 code - wallet should keep requesting until data finally received or **timeout** triggered;
- once payment data received, app prepares a transaction [see p8](https://github.com/graft-project/DesignDocuments/blob/master/RFCs/%5BRFC-003-RTVF%5D-RTA-Transaction-Validation-Flow.md) and sends it using [/pay](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#pay---process-payment) interface  
- once pay invoked - app starts polling for the payment status using [/get_payment_status](https://github.com/graft-project/GraftDocuments/blob/master/API/%5BAPI-001-SCA%5D%20Supernode%20RTA%20and%20Cryptonode%20RTA%20APIs.md#getpaymentstatus---returns-payment-status-for-given-payment-id) interface. At the same time app should implement timeout in case payment status never changed to it's final state. Once payment status changed to "Success" or one of the (Fail, RejectedByPOS) - it displays the payment status and quits. In case timeout triggered - app prints error message and quits.


### CLI interface
- _**qrcode-file**_ - filename where to read "QR" information from. Default value `wallet-qr-code.json`;   
- _**sale-timeout**_ - defines a timeout for a sale. App starts counting time once `/pay` called;    
- _**wallet-file**_ - wallet name to read wallet from (same way as `graft-wallet-cli` does);  
- _**wallet-password**_ - wallet password; 
- _**testnet**_ - 1 - tesnet, 0 - mainnet (testnet is default); 
- _**daemon-address**_ - graftnoded address in `hostname:port` form. default is `localhost:28880`;  
- _**supernode-address**_ - supernode address in `hostname:port` form. default is `localhost:28900`
