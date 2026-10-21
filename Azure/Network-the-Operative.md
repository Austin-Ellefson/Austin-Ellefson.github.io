# Network the Operative — Tracing Azure Storage Connectivity

## The Ticket

> "Our new service can reach the customer-data storage account, but I don't understand why or how. Can someone walk me through the network path?"

The application team wanted to know why its service could reach a storage account with public network access disabled, and which Azure network components carried the traffic.

## Environment

- **Platform:** Microsoft Azure
- **Resource group:** `rg-madhat-lab-chap5`
- **Access:** Reader (read-only; no configuration changes)
- **Components investigated:** Virtual network, workload subnet, network security group (NSG), effective routes, storage private endpoint, and private DNS zone
- **Method:** Trace the path using values shown in the Azure portal rather than relying on assumptions or an existing network diagram

> **Evidence status:** This draft describes the path outlined in the lab scenario. Bracketed values, effective-rule checks, and route behavior must be verified against the actual portal configuration before presenting the investigation as fully validated.

## 1. THE VNET

I began with the virtual network and identified the subnet hosting the application workload. The workload subnet uses **10.60.1.0/24**, within the VNet's private address space. This establishes the starting point for the connection to storage and the scope in which subnet-level rules and routes apply.

## 2. THE NSG

I reviewed the inbound and outbound rules on **nsg-lab-workload**, the NSG associated with the application subnet. On the inbound side, a custom rule allows HTTPS traffic on port **443** only from **203.0.113.50**, followed by a custom catch-all deny rule that blocks other inbound traffic before Azure's default rules are evaluated. On the outbound side, there are no custom rules, so traffic to the storage private endpoint is permitted by Azure's default **AllowVNetOutBound** rule. Since the storage connection stays within the VNet through the private endpoint, the NSG allows that outbound traffic without requiring an additional exception.

## 3. THE ROUTE

I reviewed the route table **rt-lab-workload** and identified a user-defined route named **force-egress**. This route sends traffic destined for `0.0.0.0/0` to a virtual appliance at **10.60.100.4**, establishing a forced-tunnel configuration for outbound traffic. However, Azure uses longest-prefix matching when selecting routes, meaning a more specific route to the storage private endpoint can take precedence over this default route. This explains how the application can reach storage through its private endpoint without sending that traffic through the firewall, while other traffic following the default route is forwarded to the virtual appliance for inspection.

## 4. THE PRIVATE ENDPOINT

I traced the storage account's private connectivity settings to its private endpoint. The endpoint's private address is **[PRIVATE ENDPOINT IP — verify in portal]** in **[ENDPOINT SUBNET — verify in portal]**. A private endpoint makes Azure Storage reachable at a private IP through Azure Private Link even with public network access disabled; the app does not need to use the storage account's public network endpoint.

## 5. THE NAME

I checked the private DNS zone and VNet link used for storage name resolution. The DNS record **[DNS RECORD / FQDN — verify in portal]** should resolve to **[PRIVATE ENDPOINT IP — verify in portal]**. This lets the app use its usual storage hostname while receiving the private endpoint address as its destination.

## The Answer — Ticket Reply

Hi Team,

I traced the network path used by the application to access the customer-data storage account. The app connects using the storage hostname, which is resolved through the linked private DNS zone to a private IP assigned to the storage private endpoint. That endpoint provides private access to Azure Storage even though public network access is disabled. The workload subnet's outbound NSG rules allow the connection, and the endpoint's more-specific /32 route takes precedence over the broader forced-tunnel route toward the firewall. As a result, this storage connection uses the private endpoint path rather than the firewall. This explains why the application can connect without public storage access.

## What I Would Flag

I would ask the networking/security team to confirm that bypassing firewall inspection for traffic to this storage private endpoint is intentional and covered by the team's security design.

## Evidence to Finalize

Before closing the investigation, record the exact portal values: workload subnet CIDR, outbound NSG rule name/priority, effective route prefix and next hop, private endpoint IP and subnet, private DNS record and its VNet link. Redact sensitive identifiers as needed before publishing screenshots.

**Key takeaway:** Read-only Azure access can be enough to reconstruct and explain a service's network path, provided every conclusion is tied to configuration evidence.
