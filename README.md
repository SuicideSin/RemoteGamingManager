# RemoteGamingManager

a [Sails](http://sailsjs.org) application.

This README is intended to serve as an architecture document for now.

Here are the rough services that this app will provide:

- *a Users service.* Manage HTTP authentication, permission control, and user
  management. Here are the models:

  - User
    - id:       uuid
    - email:    string, unique
    - password: string or whatever
    - throw on some salts, ivs, or something. ???

  - Group (this is optional, could be too complex for v1)
    - id:       uuid
    - name:     string, unique

  - GroupMemebership (this is optional, could be too complex for v1)
    - user_id:  uuid -> User
    - group_id: uuid -> Group
    - can_add_members: bool (default false)

  When we create a user, we also create a group for that user:

  ```javascript
  const user = new User({email: 'just.1.jake@gmail.com', password: 'dogs'});
  const group = new Group({name: user.email});
  group.addUser({user: user, can_add_members: false});
  ```

  If we use groups, then replace all `uuid -> User` with `uuid -> Group`

- *a ZeroTier network controller interface.*
  provide relational models that mirrors the ZeroTierOne datamodel described here:
  https://github.com/zerotier/ZeroTierOne/tree/master/service

  We will need the following models:

  - Network
    - nwid:     string(16) -> ZeroTier:Network, unique
    - owner_id: uuid -> User

    remaining fields read from the ZeroTierOne service

  - Device
    - id:        uuid
    - name:      string
    - owner_id:  uuid -> User
    - address:   string(10)

    This associates ZeroTier devices with a RemoteGamingManger user. This
    allows us to tell ZeroTier that all of a user's devices should be able
    to access that user's networks, if need be.
    You can also add other users to your networks.

  - NetworkMembership
    - nwid:      string(16)
    - user_id: uuid
    (nwid, user_id) unique

    We can provide a seperate interface to simply adding and removing
    10-digit device IDs from a network without adding *all* of a user's
    devices, or ensuring that new devices from that user can connect to the
    network.

    by "add to network" i mean specifically post to [`/controller/network/<network ID>/member/<address>`](https://github.com/zerotier/ZeroTierOne/tree/master/service#controllernetworknetwork-idmemberaddress) of the network controller to authorize a device onto the network.

    By default all networks we create should be private.

- *A AWS EC2 instance manager.*
  This is the meat and potatoes. Here are the bare minimum features:

  - Instance
    - instance_id:           aws instance id -> AWS:Instance
    - owner_id:              uuid -> User
    - device_id:             uuid -> Device
    - server_admin_password: string
    - user_admin_password:   string

    We will probably want to store a bunch of different instance state information:
    - has_steam         bool
    - has_zerotier      bool
    - currently_running bool(we can query for this)

  Here are some nice things to investigate in the AWS node api:
    - waitFor: https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/EC2.html#waitFor-property
      Waits for an EC2 instance to enter a state or a timeout. Could be useful
      to wait for startup and shutdown.
    - startInstances: this starts stopped AMI-backed instances.
      ```
      var params = {
        InstanceIds: [ /* required */
          'STRING_VALUE',
          /* more items */
        ],
        AdditionalInfo: 'STRING_VALUE',
        DryRun: true || false
      };
      ec2.startInstances(params, function(err, data) {
        if (err) console.log(err, err.stack); // an error occurred
        else     console.log(data);           // successful response
      });
      ```
    - stopInstances: pretty much the same. Instance stores are deleted, but EBS
      volumes persist attached.

    - createImage: creates an AMI from an instance.
      https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/EC2.html#createImage-property

  Here are roughly the ranking of importance of features:

    1. start/stop machine
    1. create instances using an AMI with good drivers, steam installed, and ZeroTier istalled
      1. ami has two admins: USER_ADMIN and SERVICE_ADMIN. We retain control of SERVICE_ADMIN
    1. perform some windows provisioning tasks. hopefully we can use the new Run
      Command feature for this, but the API is undocumented right now.
      1. AWS: make sure the instance has a stable IP. This might not be striclty
        necessary for ZeroTier but it seems to simplify things.
      1. Windows: scramble SERVICE_ADMIN and USER_ADMIN passwords. Save new values to DB.
        Inform the user of the admin password.
      1. ZeroTier: reset ZeroTier installation to generate new machine ID and certs.
      1. ZeroTier: join the owner's default network.
    1. built-in remote desktop: leverage [Guacamole](http://guac-dev.org/) to
      provide easy web-based remote desktop to running machines. What do we
      really need to ensure for our RDP sessions?
      1. RDP sessions must be on the admin console to enable GPU detection.
    1. Troubleshooting features. These are buttons that perform common actions
      on the cloud gaming instance:
      1. inst.changePasswordForUser(user: string, newPassword: string)
      1. inst.rebootMachine()
      1. inst.getScreenShot() could this even be possible? mite b cool
      1. lol windows networking:
        1. inst.ipconfig.release(interface: string)
        1. inst.ipconfig.renew(interface: string)
      1. zero tier controls:
        1. inst.zero.leaveNetwork() / joinNetwork()
        1. inst.zero.startService() / inst.zero.stopService()
        1. inst.zero.reset(): put everything back to square 1. reinstall?
      1. steam controls:
        1. inst.steam.startService() / stopService()
    1. security. keep each user's machine isolated, etc, encrypt relevant db
       columns, Group Policy even
