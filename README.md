## ZTNCLI

A very unimaginative name, I know.

While I think that [Key Network's ztncui](https://key-networks.com/ztncui/) is the bee's knees, it unfortunately lacks a few super important features if you are trying to *actually* use ZeroTier as a Zero Touch distributed firewall.

Specifically: it does not support editing tags, capabilities or even rules.

I did not really want to dive into their source code for something that I would also need to be able to automate so, for now, my workaround is this small Bash script that lets you manage tags and rules. And it works!

## Help

From the tool itself:

```
Usage:

    ./ztncli [options] [action]

Options:

    -c | --config <path>
        Specify path to an alternate configuration file.
        The default configuration file, in the current directory, is zt.cfg.
    -d | --details
        When listing resources, display a light level of details.
    -s | --sanity
        Check that dependencies are available and hosts reachable.
        This check takes place before moving forward with any action.

Actions:

    -h | --help
        Display this test.
    --list networks
        Display networks known to this ZeroTier moon.
    --list network clients <network name>
        Display clients registered with this ZeroTier moon.
    --list network tags <network name>
        Display tags for a given network.
    --list client tags <network name> <client name>
        Display tags for a given client in the specified network.
    --set network tag <network name> <tag number> [default number]
        Set (i.e. add or replace) tag in a network's settings.
        Or default number is omitted, it will be considered to be null.
    --unset network tag <network name> <tag number>
        Remove tag from a network's settings.
    --set client tag <network name> <client name> <tag number> <enum number>
        Set (i.e. add or replace) tag and value in a client's settings.
    --unset client tag <network name> <client name> <tag number>
        Remove tag from a client's settings.
    --list client capabilities <network name> <client name>
        Not implemented yet.
    --set client capability <network name> <client name> <...>
        Not implemented yet.
    --unset client capability <network name> <client name> <...>
        Not implemented yet.
    --list network rules <network name>
        Display network rules in compiled form.
    --set network rules <network name> [rule file name]
        Set rules for this network.
        If no file name is provided, stdin will be used.
```

Following the clever classified/clearance model documented by the ZeroTier folks, it is pretty easy to setup some on-demand RBAC. An important concept to recognize early on is that we are not going to perform actual "tag matching" but rather rely on bitwise operations, enabling us to grant multiple capabilities. In short: think rather than working with integer tag values, we are going to check permissions at a bit level.

## Example

Let's start with this simple rule set, in a file we will call `two-steps-dance.rules`:

```
tag classified
  id 1000
  enum 0 no
  enum 1 secret
  enum 2 top
  default no
;

tag clearance
  id 1001
  default 0
  flag 1 security
  flag 2 r_and_d
  flag 4 customer_support
  flag 8 sustaining
  flag 16 storage
;

# Whitelist only IPv4 (/ARP) and IPv6 traffic and allow only ZeroTier-assigned IP addresses
drop                      # drop cannot be overridden by capabilities
  not ethertype ipv4      # frame is not ipv4
  and not ethertype arp   # AND is not ARP
  and not ethertype ipv6  # AND is not ipv6
  or not chr ipauth       # OR IP addresses are not authenticated
;

break                     # reject if...
  not tor classified 0    # one end is classified
  and tand clearance 0    # no bit overlap in clearance tags
;

# Accept other packets
accept;
```

Note the use of `break` in the clearance area, leaving open the option to override it as a super user.

ztncui only "understands" compiled rules. That's what it deals with directly. So, you have two options:

1 - you can install ZeroTier's rule compiler. Node is required:

```
$ npm install --global zerotier-rule-compiler
```

and compile our rule set:

```
$ node $(npm list -g | head -1)/node_modules/zerotier-rule-compiler/cli.js |
	two-step-dance.rules > two-step-dance.compiled
```

2 - or you can work directly with the compiled rules in `two-step-dance.compiled`:

```
[
  {
    "type": "MATCH_ETHERTYPE",
    "not": true,
    "or": false,
    "etherType": 2048
  },
  {
    "type": "MATCH_ETHERTYPE",
    "not": true,
    "or": false,
    "etherType": 2054
  },
  {
    "type": "MATCH_ETHERTYPE",
    "not": true,
    "or": false,
    "etherType": 34525
  },
  {
    "type": "MATCH_CHARACTERISTICS",
    "not": true,
    "or": true,
    "mask": "1000000000000000"
  },
  {
    "type": "ACTION_DROP"
  },
  {
    "type": "MATCH_TAGS_BITWISE_OR",
    "not": true,
    "or": false,
    "id": 1000,
    "value": 0
  },
  {
    "type": "MATCH_TAGS_BITWISE_AND",
    "not": false,
    "or": false,
    "id": 1001,
    "value": 0
  },
  {
    "type": "ACTION_BREAK"
  },
  {
    "type": "ACTION_ACCEPT"
  }
]
```

Every time you modify your rules, you can apply them to ztncui's environment using:

```
$ ./ztncli --set network rules <network id> two-step-dance.compiled
# or
$ cat two-step-dance.compiled | ./ztncli --set network rules <network id>
```

Now, let's protect an asset:

```
# this asset is now going to be classified and thus not open to everyone
$ ./ztncli --set tag <network id> <asset id> 1000 1
# this asset will be acceesible to the sustaining team (8 == '00001000')
$ ./ztncli --set tag <network id> <asset id> 1001 8
```

At this point, no-one has access. Let's anoint a client:

```
# this client will have access to sustaining and security resources (9 == '00001001')
$ ./ztncli --set tag <network id> <client id> 1001 9
```

Let's verify it all look good:

```
$ ./ztncli --list network tags <network id>
Tag: 1000 Default: 1
Tag: 1001 Default: 8
$ ./ztncli --list client tags <network id> <resource id>
Tag: 1000 Value: 1
Tag: 1001 Value: 8
$ ./ztncli --list client tags <network id> <client id>
Tag: 1000 Value: 0
Tag: 1001 Value: 9
```

And now, we wait... and wait... and wait... and nothing happens.

Sorry about that. We modified ZeroTier's files but did not make it aware of that so we need to inform it. My currently brutal way of doing so is to restart the ztncui container:

```
docker exec -it zerotier bash
```

so that both the manager and the UI are updated. I am not proud of this.

And now, we wait... and wait... and eventually the rules are applied. Do not forget that the rules have to be distributed to the clients and this can take several minutes!





