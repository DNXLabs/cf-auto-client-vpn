# cf-auto-client-vpn

## What is Auto Client VPN?

AWS Client VPN costs $0.15/hr per associated subnet. It comes down to around $108/mo, and that's even if no users connected for that month. Connections and traffic are separated costs.

To reduce costs, this template associates Client VPN on a subnet once an user needs to connect.

This cloudformation creates a Client VPN configuration and a set of lambdas that automatically:
- Associate to a subnet when an user needs to connect
- Dissassociate it from the subnet when there are no connections

Which this technique, Client VPN cost can reduce significantly, as now you only pay when in use.

## How it works?

To connect:
1. User calls a URL
2. User waits 5 minutes for VPN to come up
3. User tries to connect, connection is stablished.

Upon disconnection:
1. 60 minutes later, VPN is automatically down

## Architecture

This cloudformation template has a parameter *VPNUp* that controls the association to the subnet passed.

### Lambda AutoStart

This lambda has a URL (see the cloudformation stack Output for URL) that when called, changes the parameter *VPNUp* of the stack to true.

### Lambda AutoStop

This lambda runs every 10 minutes and changes the parameter *VPNUp* of the stack to false. Based on the following conditions:
- There were no users connected for the last 60 minutes (connections linger for 60 minutes after disconnection)
- There has been more than 60 minutes from when VPNUp was set to true. - This is to avoid VPN going down right after it was associated (VPNUp set to true), before any users had a chance to connect.

The second condition is controlled by an SSM parameter that records the timestamp of the last VPNUp=true event.

## How to use

This template configures AWS Client VPN to use with a SAML provider (usually IAM Identity Center/SSO).

Follow the instructions here to create the IAM Identity Providers: [https://aws.amazon.com/blogs/security/authenticate-aws-client-vpn-users-with-aws-single-sign-on/](https://aws.amazon.com/blogs/security/authenticate-aws-client-vpn-users-with-aws-single-sign-on/)

Go to cloudformation and Create Stack.

Enter the parameters:

- `SplitTunnel` - `true` will split the traffic between VPC and internet, `false` means all traffic to the internet goes throught the VPN - usually this option is used for privacy reasons.
- `SAMLProviderArn` and `SelfServiceSAMLProviderArn` from the IdPs created from the blog post above.
- `VpdId` and `SubnetId` is the VPC and Subnet to associate this subnet (when Up)
- `VPNUp` keep this `false` unless you want to manually associate the VPN to the subnet.
