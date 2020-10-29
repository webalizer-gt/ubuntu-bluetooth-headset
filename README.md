# ubuntu-bluetooth-headset
How to use Bluetooth Headsets (headphones AND microphone) in Ubuntu/Mint 20.x

Some earbuds profiles are A2DP, AVRCP and HFP but not HSP. Pulseaudio only supports HSP out-of-the-box. In order to make HSP/HFP work, you have to enable HFP on pulseaudio which needs ofono. This also applies for Apple Airpods. Here is how to do it on Ubuntu/Mint 20.x:

#### 1. Install ofono:
    sudo apt install ofono
#### 2. Configure pulseaudio to use ofono:
- Goto `/etc/pulse/default.pa`  
- Find the line `load-module module-bluetooth-discover` and change it in `load-module module-bluetooth-discover headset=ofono`
- Add the user pulse to group bluetooth to grant the permission:  
`sudo usermod -aG bluetooth pulse` (probably it's already correct)  
VERY IMPORTANT: To grant the permission, add this to `/etc/dbus-1/system.d/ofono.conf` (before `</busconfig>`):

        <policy user="pulse">  
          <allow send_destination="org.ofono"/>  
        </policy>

#### 3. Provide `phonesim` to `ofono`. In order to make ofono work, you have to provide a modem to it! You can install a modem emulator called phonesim (implemented by ofono) to make it work:
The package was removed from Debian Unstable at 2019-09-13, so it will not be until official port to Qt5.
- You can install it from some third-party PPA:

        sudo add-apt-repository ppa:smoser/bluetooth
        sudo apt-get install ofono-phonesim
- Configure phonesim by adding the following lines to `/etc/ofono/phonesim.conf`:

        [phonesim]
        Driver=phonesim
        Address=127.0.0.1
        Port=12345
        
#### 4. Manual mode  
Now your system setup is completed for a manual usage. To use it you need to start ofono-phonesim and then restart the Bluetooth stack. To do so:

    ofono-phonesim -p 12345 /usr/share/phonesim/default.xml & sudo service ofono restart
    /usr/share/ofono/scripts/enable-modem /phonesim
    sudo service bluetooth restart
As a test, you can run `/usr/share/ofono/scripts/list-modems` and should see the phonesim modem initialized.  
Now, open your Bluetooth settings, connect your Bluetooth device and switch it to HSP/HFP Profile.
#### 5. Automating setup
If your manual mode works, you are almost done. It is boring to execute the steps above every time, so I decided to tune my system startup since I need this feature every day.

Let’s start with `phonesim`. Create a systemd unit file `/etc/systemd/system/phonesim.service` with the following content. This will start `phonesim` on start of the system.

    [Unit]
        Description = Phonesim Control Daemon
        After = network.target network-online.target dbus.service
        Wants = network-online.target
        Requires = dbus.service
  
    [Service]
        Type = simple
        ExecStart = /usr/bin/ofono-phonesim -p 12345 /usr/share/phonesim/default.xml
        Restart = on-abort
        StartLimitInterval = 60
        StartLimitBurst = 10
  
    [Install]
        WantedBy = multi-user.target
If you changed the port in your modem definition, please change it accordingly. Please make sure, the line `ExecStart` is joined.

The next service is `ofono`. We need to tell systemd that `ofono.service` should depend on `phonesim.service`. To do so create unit file `/etc/systemd/system/ofono.service` with the following content:

        [Unit]
            Description=Telephony service
            After = phonesim.service
            Requires = phonesim.service


        [Service]
            Type=dbus
            BusName=org.ofono
            ExecStart=/usr/sbin/ofonod -n
            StandardError=null

        [Install]
            WantedBy=multi-user.target
The last step is to enable the `phonesim` modem. To do so, create a new unit file `/etc/systemd/system/ofono-enable-modem.service` with the following content:

    [Unit]
        Description=Enables Phonesim Modem
        After = ofono.service
        Requires = ofono.service


    [Service]
        Type=oneshot
        ExecStart=/usr/share/ofono/scripts/enable-modem /phonesim

    [Install]
        WantedBy=multi-user.target
This service will be run just after the `ofono.service` and enable our modem on boot.

Now, we need to make sure the provided services are activated on boot. Please reload your systemd and enable them by running:

    sudo systemctl daemon-reload
    sudo systemctl enable phonesim
    sudo systemctl enable ofono
    sudo systemctl enable ofono-modem
That’s it. Now reboot and check if the modem is connected directly after the boot by running `/usr/share/ofono/scripts/list-modems`. You should see something like this:

    [/phonesim ]
        Online = 0
        Powered = 1
        Lockdown = 0
        Emergency = 0
        Manufacturer = MeeGo
        Model = Synthetic Device
        Revision = REV1
        Serial = 1234567890
        Interfaces = org.ofono.SmartMessaging org.ofono.PushNotification org.ofono.MessageManager org.ofono.Phonebook org.ofono.TextTelephony org.ofono.RadioSettings org.ofono.CallForwarding org.ofono.SimToolkit org.ofono.SimAuthentication org.ofono.AllowedAccessPoints org.ofono.VoiceCallManager org.ofono.SimManager 
        Features = sms tty rat stk sim 
        Type = hardware
    ...

Happy conferencing…

References:

https://askubuntu.com/questions/831331/failed-to-change-profile-to-headset-head-unit/1236379#1236379  
https://askubuntu.com/questions/1242450/when-will-add-ofono-phomesim-to-the-focal-repository-20-04
https://zambrovski.medium.com/using-bluetooth-headset-on-ubuntu-790ce6eecc2  
https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Bluetooth/  
https://stackoverflow.com/a/51697709/692422  
https://docs.plasma-mobile.org/Ofono.html
