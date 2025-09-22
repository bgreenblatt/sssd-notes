# sssd-notes

## SSSD:

Github: https://github.com/SSSD/sssd

As far as I can tell, it doesn't do any translation when we call sss_nss_getsidbyid so if I am right, there is no need to install and configure SSSD on the dmns. 
Each domain is added to the SSSD in memory list with a range. There is no requirement that the range should begin or end at some interval that would make the last digits of the UID match the RID of the SID. For example, the range size could be some large prime number.

So, unless we know the range specified on the client side, we don't have any way of turning the UID that we have received into an RID that we can use to look up the SID in AD. So, for each tenant domain, we need to know the values of the configuration parameters below.


## SSSD/AD Config params: https://linux.die.net/man/5/sssd-ad

### Advanced Configuration

#### ldap_idmap_range_min (integer)
Specifies the lower bound of the range of POSIX IDs to use for mapping Active Directory user and group SIDs.
NOTE: This option is different from "min_id" in that "min_id" acts to filter the output of requests to this domain, whereas this option controls the range of ID assignment. This is a subtle distinction, but the good general advice would be to have "min_id" be less-than or equal to "ldap_idmap_range_min"

Default: 200000

#### ldap_idmap_range_max (integer)
Specifies the upper bound of the range of POSIX IDs to use for mapping Active Directory user and group SIDs.
NOTE: This option is different from "max_id" in that "max_id" acts to filter the output of requests to this domain, whereas this option controls the range of ID assignment. This is a subtle distinction, but the good general advice would be to have "max_id" be greater-than or equal to "ldap_idmap_range_max"

Default: 2000200000

#### ldap_idmap_range_size (integer)
Specifies the number of IDs available for each slice. If the range size does not divide evenly into the min and max values, it will create as many complete slices as it can.
Default: 200000

#### ldap_idmap_default_domain_sid (string)
Specify the domain SID of the default domain. This will guarantee that this domain will always be assigned to slice zero in the ID map, bypassing the murmurhash algorithm described above.
Default: not set

The range of ids is divided into slices based on the range size. To find the right slice for a domain, the domain sid is hashed as follows. The SID string is passed through the murmurhash3 algorithm to convert it to a 32-bit hashed value. Then take the modulus of this value with the total number of available slices to pick the slice. In the default values above, there are 10,000 slices with each slice having 200,000 uids available. So, if we have the uid: 1050001946 with the default values, then that means it is really uid 1946 in the domain that hashed to the 5,250th slice. In the SSSD source code, to find the right domain for a UID, the code iterates through a saved list of ranges and finds the one that matches the UID.

So, to make the existing SSSD code work, we have to add the domains to the SSSD framework by using a call such as: sss_idmap_add_domain_ex. Each vStore will have its own internal SSSD context that it will use when making these calls

But, this is a potential problem since two different customer domains could hash to the same slice. The SSSD-AD man page says this: "NOTE: It is possible to encounter collisions in the hash and subsequent modulus. In these situations, we will select the next available slice, but it may not be possible to reproduce the same exact set of slices on other machines (since the order that they are encountered will determine their slice). In this situation, it is recommended to either switch to using explicit POSIX attributes in Active Directory (disabling ID-mapping) or configure a default domain to guarantee that at least one is always consistent. See "Configuration" for details." This is probably an edge case that we don't need to worry about.

The existing code that makes sssd API calls doesn't use any of the idmap.json configuration, so it seems that it is using SSSD CLI's to decide which LDAP/AD server to use to do lookups. But since no SSSD is configured on the cluster I think it will fail.
