---
layout: post
title: Secure Financial Messaging with JBoss Fuse & A-MQ
date: 2016-05-24
comments: True
categories:
---

Exchanging financial messages securely within an organization is challenging.  Although financial back office systems often lie within a trusted zone, we cannot be complacent about message security - especially if we’re sending instructions to move potentially large amounts of money.  But there are many ways we can tackle securing messaging within the back office environment.  Using a message broker like ActiveMQ is a good start, since it provides asynchronous, guaranteed and secure message delivery whilst eliminating data at rest.  But is this enough to prevent our sensitive messages being compromised?

Let’s have a closer look at the typical types of messages that are exchanged by financial institutions.  Here is a sample message:

~~~ Text
{1:F01ABCBUAU0XXXX0000000000}{2:I103DEFHUS33XXXXN}{4:
:20:TEST1
:23B:CRED
:32A:131231USD111111111111,11
:50K:/0123456789012345678901234567890123
Joe Blogs
7 Times Square
Level 45
New York NY 10036 USA
:59:/0123456789012345678901234567890123
John Smith
123 Times Square
New York NY 10036
United States of America
:70:Wire transfer for services rendered
:71A:OUR
-}
~~~

The above message is a payment instruction to move money from Joe Blogs account to John Smith’s account.  The sensitive data we want to protect is primarily the account numbers and beneficiary details.  But lets take a step back for a moment and consider some other issues with this message.  They include

* Proprietary format
* The message standard changes annually
* Domain specific
* Contains complex business rules
* The data is non-descriptive

So how can we solve these problems and why do we care when we’re talking message security?  The answer is many fold:

1. Our priority to protect message integrity.  We can’t have a hacker changing account numbers and beneficiary details.
2. Do we care if a hacker sees this information while it’s being transferred?  Probably not…
3. We need somewhere to store a digital signature - some kind of hash of the payload based on a shared secret. This ensures integrity of the message.

The way we solve some of these problems is by using a classic Enterprise Integration Pattern (EIP) - the Canonical Data Model.  Basically, a common (and open) language that we define and control to speak system to system.  Here is how I would share the above message with internal systems in my organization:

~~~ XML
<?xml version="1.0" encoding="UTF-8"?>
<!-- Version 1.0 -->
<paymentInstructionMessage xmlns="http://ABCBank.Gateway.Contract/paymentInstructionMessage/v1.0"
		xmlns:ns0="http://ABCBank.Gateway.Contract/header/v1.0" 
		xmlns:ns1="http://ABCBank.Gateway.Contract/paymentInstruction/v1.0" 
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://ABCBank.Gateway.Contract/paymentInstructionMessage/v1.0 paymentInstructionMessage.xsd">
	<header>
		<ns0:serviceVersion>v11.11</ns0:serviceVersion>
		<ns0:environment>dev</ns0:environment>
		<ns0:transactionID>1234</ns0:transactionID>
		<ns0:messageCreationDateTime>2001-12-17T09:30:47Z</ns0:messageCreationDateTime>
		<ns0:priority>Normal</ns0:priority>
		<ns0:possibleDuplicate>T</ns0:possibleDuplicate>
		<ns0:sendingSystemComponent>SAA</ns0:sendingSystemComponent>
		<ns0:destinationSystemComponent>APOLLO</ns0:destinationSystemComponent>
	</header>
	<paymentInstruction>
		<ns1:DestinationBIC>DEFHUS33XXX</ns1:DestinationBIC>
		<ns1:MessageReferenceNumber>TEST1</ns1:MessageReferenceNumber>
		<ns1:ValueDate>131231</ns1:ValueDate>
		<ns1:CurrencyCode>USD</ns1:CurrencyCode>
		<ns1:Amount>111111111111,11</ns1:Amount>
		<ns1:OrderingCustomerAccount>0123456789012345678901234567890123</ns1:OrderingCustomerAccount>
		<ns1:OrderingCustomerName>Joe Blogs</ns1:OrderingCustomerName>
		<ns1:OrderingCustomerAddress1>7 Times Square</ns1:OrderingCustomerAddress1>
		<ns1:OrderingCustomerAddress2>Level 45</ns1:OrderingCustomerAddress2>
		<ns1:OrderingCustomerAddress3>New York NY 10036 USA</ns1:OrderingCustomerAddress3>
		<ns1:IntermediaryInstitution></ns1:IntermediaryInstitution>
		<ns1:IntermediaryInstitutionLocalClearingSystem></ns1:IntermediaryInstitutionLocalClearingSystem>
		<ns1:IntermediaryInstitutionName></ns1:IntermediaryInstitutionName>
		<ns1:AccountWithInstitution></ns1:AccountWithInstitution>
		<ns1:AccountWithInstitutionLocalClearingSystem></ns1:AccountWithInstitutionLocalClearingSystem>
		<ns1:AccountWithInstitutionName></ns1:AccountWithInstitutionName>
		<ns1:BeneficiaryCustomerAccount>0123456789012345678901234567890123</ns1:BeneficiaryCustomerAccount>
		<ns1:BeneficiaryCustomerName>John Smith</ns1:BeneficiaryCustomerName>
		<ns1:BeneficiaryCustomerAddress1>123 Times Square</ns1:BeneficiaryCustomerAddress1>
		<ns1:BeneficiaryCustomerAddress2>New York NY 10036</ns1:BeneficiaryCustomerAddress2>
		<ns1:BeneficiaryCustomerAddress3>United States of America</ns1:BeneficiaryCustomerAddress3>
		<ns1:RemittanceInformation>Wire transfer for services rendered</ns1:RemittanceInformation>
		<ns1:DetailofCharges>OUR</ns1:DetailofCharges>
	</paymentInstruction>
</paymentInstructionMessage>
~~~

The beauty behind the above structure is that it’s:

* Self describing
* Human readable
* Generic structure
* Open
* Can easily be validated against an XSD
* Separates business data from system metadata (contained in the header block)

But how does this improve message security? The fact is it doesn’t by itself, but it does give us a platform to add a Digital Signature or transform into a more RESTful structure like JSON.  This is how we solve the message integrity issue - protecting account numbers and beneficiary details from modification during transmission.

Lets talk some more about the Digital Signature.  The easiest way to achieve this is using a hash algorithm.  The one I like to use the HMAC-SHA256 (described [here](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code)).  By using a shared secret (password) between your app and back office system, we can generate a hash and include it either as a JMS property, HTTP header or as part of an S/MIME message.  The code we use looks something like this:

~~~ Java
/**
 * Calculate the LAU (digital signature)
 * 
 * @param payload the raw payload (in bytes)
 * @return the LAU signature
 * @throws Exception
 */
private byte[] calculateLAU(byte[] payload) throws Exception {
    Mac m = Mac.getInstance("HmacSHA256");

    // initialize key with shared secret from SAA
    SecretKeySpec keyspec = new SecretKeySpec(this.hmacKey.getBytes(Charset.forName("US-ASCII")), "HmacSHA256");
    m.init(keyspec);
    
    // calculate the LAU
    byte[] lau = m.doFinal(payload);
    byte[] lau_to_encode = new byte[16];
    
     System.arraycopy(lau, 0, lau_to_encode, 0, 16);
     LOGGER.info("LAU value is: [" + Base64.encodeBase64String(lau_to_encode) + "]");
     return Base64.encodeBase64(lau_to_encode);
}
~~~

To preserve whitespace and CRLF, we base64 encode the payload before generating the hash, adding it to the JMS properties and transmitting it.  When the receiving system receives the JMS message, it must first generate a hash based on the base64 encoded payload and match that with the Digital Signature JMS property.  If the generated hash doesn’t match the JMS property, we know our message has been tampered with and must be discarded.

But lets step back from the weeds and take a 30,000 foot view of our flow using the EIP lexicon:

![EIP End to End Flow](/images/eip_flow_end_to_end.png)

To achieve a secure channel between our back office and (proprietary) messaging gateway, we have:

1. A-MQ providing Guaranteed Delivery, eliminating data at rest and providing asynchronous “fire and forget” messaging
2. The Fuse JMS client providing a Messaging Gateway
3. Using our Digital Signature and Canonical Data Model above, we have an implementation of the Envelope Wrapper EIP
4. Using a message transformer, we can transform the message to/from our Canonical Data Model to the proprietary message format accepted by the Messaging Gateway
5. The Channel Adapter step takes care of preparing the message to be sent / received over the external Messaging Gateway

You’ll notice I haven’t mentioned SSL once during this article.  That’s because it’s not really required for what we’re trying to ultimately achieve: guaranteeing message integrity.  SSL gives us authentication, authorization and an encrypted channel but it doesn’t provide us with a means to guarantee message integrity, therefore unless your organization mandates all back office communication channels must use SSL, I wouldn’t bother given the effort to administer and manage certificates for each system.
