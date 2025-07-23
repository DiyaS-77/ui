import dbus
import dbus.service
import dbus.mainloop.glib
import os
import subprocess
import time
from gi.repository import GObject
import mimetypes
from dbus.mainloop.glib import DBusGMainLoop

# Set the D-Bus main loop
dbus.mainloop.glib.DBusGMainLoop(set_as_default=True)


class BluetoothDeviceManager:
    """
    A class for managing Bluetooth devices using the BlueZ D-Bus API.

    This manager provides capabilities for discovering, pairing, connecting,
    streaming audio (A2DP), media control (AVRCP), and removing Bluetooth devices.
    """

    def __init__(self):
        """
        Initialize the BluetoothDeviceManager by setting up the system bus and adapter.
        """
        self.bus = dbus.SystemBus()
        self.adapter_path = '/org/bluez/hci0'
        self.adapter_proxy = self.bus.get_object('org.bluez', self.adapter_path)
        self.adapter = dbus.Interface(self.adapter_proxy, 'org.bluez.Adapter1')
        self.stream_process = None
        self.device_path = None
        self.device_address = None
        self.device_sink = None
        self.devices = {}
        self.last_session_path = None
        self.opp_process = None

    def start_discovery(self):
        """
        Start scanning for nearby Bluetooth devices.
        """
        self.adapter.StartDiscovery()

    def stop_discovery(self):
        """
        Stop Bluetooth device discovery.
        """
        self.adapter.StopDiscovery()

    def power_on_adapter(self):
        """
        Power on the local Bluetooth adapter.
        """
        adapter = dbus.Interface(
            self.bus.get_object("org.bluez", self.adapter_path),
            "org.freedesktop.DBus.Properties"
        )
        adapter.Set("org.bluez.Adapter1", "Powered", dbus.Boolean(True))

    def inquiry(self, timeout):
        """
        Scan for nearby Bluetooth devices for a specified duration.

        :param timeout: Duration in seconds to scan for devices.
        :return: List of discovered devices in the format "Alias (Address)".
        """
        self.start_discovery()
        time.sleep(timeout)
        self.stop_discovery()

        discovered = []
        om = dbus.Interface(self.bus.get_object("org.bluez", "/"), "org.freedesktop.DBus.ObjectManager")
        objects = om.GetManagedObjects()
        for path, interfaces in objects.items():
            if "org.bluez.Device1" in interfaces:
                device_props = dbus.Interface(self.bus.get_object("org.bluez", path),
                                              dbus_interface="org.freedesktop.DBus.Properties")
                try:
                    address = device_props.Get("org.bluez.Device1", "Address")
                    alias = device_props.Get("org.bluez.Device1", "Alias")
                    discovered.append(f"{alias} ({address})")
                except:
                    continue
        return discovered

    def _get_device_path(self, address):
        """
        Format the Bluetooth address to get the BlueZ D-Bus object path.

        :param address: Bluetooth device MAC address.
        :return: D-Bus object path.
        """
        formatted_address = address.replace(":", "_")
        return f"/org/bluez/hci0/dev_{formatted_address}"

    def find_device_path(self, address):
        """
        Find the full D-Bus object path of a device by its Bluetooth address.

        :param address: Bluetooth device MAC address.
        :return: D-Bus object path or None if not found.
        """
        om = dbus.Interface(self.bus.get_object("org.bluez", "/"), "org.freedesktop.DBus.ObjectManager")
        objects = om.GetManagedObjects()
        for path, interfaces in objects.items():
            if "org.bluez.Device1" in interfaces:
                props = interfaces["org.bluez.Device1"]
                if props.get("Address") == address:
                    return path
        return None

    def _get_device_interface(self, device_path):
        """
        Get the org.bluez.Device1 interface for the specified device path.

        :param device_path: D-Bus object path of the device.
        :return: DBus Interface for the device.
        """
        return dbus.Interface(
            self.bus.get_object("org.bluez", device_path),
            "org.bluez.Device1"
        )

    def pair(self, address):
        """
        Pair with the device at the given Bluetooth address.

        :param address: Bluetooth device MAC address.
        :return: True if paired successfully, False otherwise.
        """
        try:
            device_path = self._get_device_path(address)
            device = self._get_device_interface(device_path)

            print(f"Initiating pairing with {device_path}")
            device.Pair()
            return True

        except dbus.exceptions.DBusException as e:
            if "Did not receive a reply" in str(e) or "Timeout" in str(e):
                print("[BluetoothDeviceManager] Pair() timeout, checking status...")
                time.sleep(2)

                device_props = dbus.Interface(
                    self.bus.get_object("org.bluez", device_path),
                    "org.freedesktop.DBus.Properties"
                )
                paired = device_props.Get("org.bluez.Device1", "Paired")
                if paired:
                    print("[BluetoothDeviceManager] Device is actually paired.")
                    return True

            print(f"[BluetoothDeviceManager] Pairing failed: {e}")
            return False

    def br_edr_connect(self, address):
        """
        Establish a BR/EDR connection to the specified Bluetooth device.

        :param address: Bluetooth device MAC address.
        :return: True if connected, False otherwise.
        """
        device_path = self.find_device_path(address)
        if device_path:
            try:
                device = dbus.Interface(self.bus.get_object("org.bluez", device_path),
                                        dbus_interface="org.bluez.Device1")
                device.Connect()

                props = dbus.Interface(self.bus.get_object("org.bluez", device_path),
                                       "org.freedesktop.DBus.Properties")
                connected = props.Get("org.bluez.Device1", "Connected")
                if connected:
                    print("Connection is successful")
                    return True
                else:
                    print("Connection attempted but not confirmed")
                    return False
            except Exception as e:
                print(f"Connection failed: {e}")
                return False
        else:
            print("Device path not found for connection")
            return False

    def disconnect_le_device(self, address):
        """
        Disconnect a Bluetooth LE device using BlueZ D-Bus interface.

        :param address: Bluetooth MAC address (e.g., 'C0:26:DA:00:12:34')
        :return: True if disconnect successful or not connected, False otherwise.
        """
        try:
            device_path = self._get_device_path(address)

            # Access device and its properties
            device = dbus.Interface(self.bus.get_object("org.bluez", device_path), "org.bluez.Device1")
            props = dbus.Interface(self.bus.get_object("org.bluez", device_path), "org.freedesktop.DBus.Properties")

            # Check if already disconnected
            connected = props.Get("org.bluez.Device1", "Connected")
            if not connected:
                print(f"[BluetoothDeviceManager] Device {address} is already disconnected.")
                return True

            # Perform disconnect
            print(f"[BluetoothDeviceManager] Disconnecting device {address}...")
            device.Disconnect()
            time.sleep(1)  # Optional: allow async operations to complete

            print(f"[BluetoothDeviceManager] Device {address} disconnected successfully.")
            return True

        except dbus.exceptions.DBusException as e:
            print(f"[BluetoothDeviceManager] Error disconnecting device {address}: {e}")
            return False

    def remove_device(self, address):
        """
        Remove the bonded device from the system.

        :param address: Bluetooth device MAC address.
        :return: True if removed or already gone, False otherwise.
        """
        obj = self.bus.get_object("org.bluez", "/")
        manager = dbus.Interface(obj, "org.freedesktop.DBus.ObjectManager")
        objects = manager.GetManagedObjects()

        for path, interfaces in objects.items():
            if "org.bluez.Device1" in interfaces:
                if interfaces["org.bluez.Device1"].get("Address") == address:
                    print(f"[BluetoothDeviceManager] Removing device {path}")
                    try:
                        adapter = dbus.Interface(
                            self.bus.get_object("org.bluez", self.adapter_path),
                            "org.bluez.Adapter1"
                        )
                        adapter.RemoveDevice(path)
                        return True
                    except dbus.exceptions.DBusException as e:
                        if "org.freedesktop.DBus.Error.UnknownObject" in str(e):
                            print(f"[BluetoothDeviceManager] Device {address} already removed")
                            return True  # Still a success
                        else:
                            print(f"[BluetoothDeviceManager] Failed to remove {address}: {e}")
                            return False

        print(f"[BluetoothDeviceManager] Device with address {address} not found")
        return True  # Treat as success since it's already not present

    def is_a2dp_streaming(self) -> bool:
        """
        Check if an A2DP stream is currently active using PulseAudio.

        Returns:
            bool: True if audio is streaming to a Bluetooth A2DP sink, False otherwise.
        """

        try:
            # Get all active sink inputs (audio streams)
            output = subprocess.check_output("pactl list sink-inputs", shell=True, text=True)

            # Check if any sink input is directed to a Bluetooth A2DP sink
            if "bluez_sink" in output:
                return True

            return False

        except subprocess.CalledProcessError:
            # pactl command failed
            return False

    '''def start_a2dp_stream(self, address, filepath=None):
        """
        Start streaming audio to a Bluetooth A2DP device.

        :param address: Bluetooth device MAC address.
        :param filepath: Path to the audio file to stream (WAV format).
        :return: Status message.
        """
        device_path = self.find_device_path(address)
        if not device_path:
            return "Device not found"

        try:
            device = dbus.Interface(self.bus.get_object("org.bluez", device_path), "org.bluez.Device1")
            device.Connect()
            print(f"[A2DP] Connected to {address}")

            if not filepath:
                return "No audio file specified for streaming"

            self.stream_process = subprocess.Popen(
                ["aplay", filepath],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )
            return f"Streaming started with {filepath}"

        except Exception as e:
            return f"A2DP stream error: {str(e)}"'''

    def start_a2dp_stream(self, address, filepath=None):
        device_path = self.find_device_path(address)
        if not device_path:
            return "Device not found"
        try:
            device = dbus.Interface(self.bus.get_object("org.bluez", device_path), "org.bluez.Device1")
            props = dbus.Interface(self.bus.get_object("org.bluez", device_path), "org.freedesktop.DBus.Properties")
            connected = props.Get("org.bluez.Device1", "Connected")
            if not connected:
                device.Connect()
                time.sleep(1.5)
            print(f"[A2DP] Connected to {address}")
            if not filepath:
                return "No audio file specified for streaming"
            self.stream_process = subprocess.Popen(
                ["aplay", filepath],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )
            return f"Streaming started with {filepath}"
        except Exception as e:
            return f"A2DP stream error: {str(e)}"

    def stop_a2dp_stream(self):
        """
        Stop the current A2DP audio stream.

        :return: Status message.
        """
        if self.stream_process:
            self.stream_process.terminate()
            self.stream_process = None
            return "A2DP stream stopped"
        return "No active A2DP stream"

    def media_control(self, command):
        """
        Send an AVRCP media control command to a connected A2DP device.

        Supported commands: play, pause, next, previous, rewind.

        :param command: The command to send as a string.
        :return: Result message.
        """
        valid = {
            "play": "Play",
            "pause": "Pause",
            "next": "Next",
            "previous": "Previous",
            "rewind": "FastRewind"
        }

        if command not in valid:
            return f"Invalid command: {command}"

        om = dbus.Interface(self.bus.get_object("org.bluez", "/"), "org.freedesktop.DBus.ObjectManager")
        objects = om.GetManagedObjects()

        for path, interfaces in objects.items():
            if "org.bluez.MediaControl1" in interfaces:
                try:
                    control_iface = dbus.Interface(self.bus.get_object("org.bluez", path), "org.bluez.MediaControl1")
                    getattr(control_iface, valid[command])()
                    return f"AVRCP {command} sent to {path}"
                except Exception as e:
                    return f"Error sending AVRCP {command}: {str(e)}"
        return "No MediaControl1 interface found (is device connected as A2DP Source with AVRCP?)"
