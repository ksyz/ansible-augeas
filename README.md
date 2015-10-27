ansible-augeas
==============

Augeas module which exposes simple API for `match`, `set`, `rm` and `ins` operations. Additionally it allows lens loading through `transform` operation. You can execute commands one by one or in chunk (some of them are probably sensible only in bulk mode e.g. `ins` and `transform`).

Requirements:
  - augeas
  - augeas python bindings

Options:
  - `command`
      - required: when `commands` is not used
      - choices: [`set`, `ins`, `rm`, `match`, `transform`]
      - description:

         Whether given path should be modified, inserted (ins command can be really used in multicommand mode), removed or matched.

        Command "match" passes results through "result" attribute - every item on this list is an object with "label" and "value" (check second example below). Other commands returns true in case of any modification (so this value is always equal to "changed" attribue - this make more sens in case of bulk execution)

        Every augeas action is a separate augeas session, so `ins` command has probably only sens in bulk mode (when command=`commands`)

  - `path`:
      - required: when any `command` is used
      - description: Variable path.
  - `value`:
      - required: when `command = set`
      - description: Variable value.
  - `label`:
      - required: when `command = ins`
      - description: Label for new node.
  - `where`:
      - required: false
      - default: 'after'
      - choices: [`before`, `after`]
      - description: Position of node insertion against given `path`.
  - `commands`
      - required: when `command` is not used
      - description: Execute many commands at once (some configuration entries have to be created/updated at once - it is impossible to split them across multiple "set" calls). Standard shell quoting is allowed (rember to escape all quotes inside pahts/values - check last example). Expected formats: "set PATH VALUE", "rm PATH" or "match PATH" (look into examples for more details). You can separate commands with any white characters (new lines, spaces etc.). Result is passed through `result` attribute and contains list of tuples: (command, command result).
  - `root`:
      - required: false
      - description: The filesystem root - config files are searched realtively to this path (fallbacks to `AUGEAS_ROOT` which fallbacks to  `/`).
  - `loadpath`:
      - required: false
      - description: Colon-spearated list of directories that modules should be searched in.

Examples:

  - Simple value change

        - name: Force password on sudo group
          action: augeas path=/files/etc/sudoers/spec[user=\"sudo\"]/host_group/command/tag value=PASSWD

  - Results of `match` command - given below action:

        - name: Fetch sshd allowed users
          action: augeas command="match" path="/files/etc/ssh/sshd_config/AllowUsers/*"
          register: ssh_allowed_users

      you can expect value of this shape to be set in `ssh_allowed_users` variable:

        {"changed": false,
         "result": [{"label": "/files/etc/ssh/sshd_config/AllowUsers/1",
                     "value": "paluh"},
                    {"label": "/files/etc/ssh/sshd_config/AllowUsers/2",
                     "value": "foo"},
                    {"label": "/files/etc/ssh/sshd_config/AllowUsers/3",
                     "value": "bar"}]}

  - Quite complex modification - fetch values lists and append new value only if it doesn't exists already in config

        - name: Check whether given user is listed in sshd_config
          action: augeas command='match' path="/files/etc/ssh/sshd_config/AllowUsers/*[.=\"{{ user }}\"]"
          register: user_entry
        - name: Allow user to login through ssh
          action: augeas command="set" path="/files/etc/ssh/sshd_config/AllowUsers/01" value="{{ user }}"
          when: "user_entry.result|length == 0"

  - Bulk command execution

        - name: Add piggy to /etc/hosts
          action:  augeas commands='set /files/etc/hosts/01/ipaddr 192.168.0.1
                                    set /files/etc/hosts/01/canonical pigiron.example.com
                                    set /files/etc/hosts/01/alias[1] pigiron
                                    set /files/etc/hosts/01/alias[2] piggy'
  - Transform example

        - name: Modify sshd_config in custom location
          action: augeas commands='transform "sshd" "incl" "/home/paluh/programming/ansible/tests/sshd_config"
                                   set "/files/home/paluh/programming/ansible/tests/sshd_config/AllowUsers" "paluh"'

  - Insert example

        - name: Turn on ssh agent forwarding
          action: augeas commands='ins ForwardAgent before "/files/etc/ssh/sshd_config"
                                   set "/files/etc/ssh/sshd_config/ForwardAgent" "yes"'

  - Correct quoting in commands expressions (augeas requires quotes in path matching expressions: iface[.=\"eth0\"])

        - name: Redefine eth0 interface
          action: augeas commands='rm /files/etc/network/interfaces/iface[.=\"eth0\"]
                                   set /files/etc/network/interfaces/iface[last()+1] eth0
                                   set /files/etc/network/interfaces/iface[.=\"eth0\"]/family inet
                                   set /files/etc/network/interfaces/iface[.=\"eth0\"]/method manual
                                   set /files/etc/network/interfaces/iface[.=\"eth0\"]/pre-up "ifconfig $IFACE up"
                                   set /files/etc/network/interfaces/iface[.=\"eth0\"]/pre-down "ifconfig $IFACE down"'

Debugging:

  - If you want to check files which are accessible by augeas on server just run:

        ansible all -u USERNAME -i INVENTORY_FILE -m augeas -a \'command="match" path="/augeas/files//*"

  - In case of any errors during augeas execution of your operations this module will return content of `/augeas//error` and you should be able to find problems related to your actions
