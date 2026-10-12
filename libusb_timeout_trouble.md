# `libusb_interrupt_transfer` timeout trouble

`libusb_interrupt_transfer` kept giving me invalid data despite returning a success for the following code:

```cpp
#define EVT_TIMEOUT 5

int main() {
    libusb_context *ctx;
    auto res = libusb_init(&ctx);
    assert(res == LIBUSB_SUCCESS);
    // BT Dev
    auto dev = libusb_open_device_with_vid_pid(ctx, BT_USB_DEV_VID, BT_USB_DEV_PID);
    if (!dev) {
        std::println("Failed to open device");
        return 1;
    }

    res = libusb_set_auto_detach_kernel_driver(dev, true);
    assert(res == LIBUSB_SUCCESS);
    res = libusb_claim_interface(dev, 0);
    assert(res == LIBUSB_SUCCESS);

    uint8_t buffer[512];
    auto writer = hci_cmd_writer(buffer, make_opcode(0x01, 0x0019));
    const uint8_t addr[6] = BT_REMOTE_ADDR;
    writer.write_bytes(0x00, addr, sizeof(addr));
    writer.write_u8(0x06, 0x00);
    writer.write_u8(0x07, 2);
    writer.write_u16le(0x08, 0x00);
    writer.set_body_size(0xA);

    res = libusb_control_transfer(dev, 0x20, 0x00, 0x00, 0x00, writer.packet_data(), writer.packet_size(), 3000);
    if (res < 0)
        print_err_and_exit(res);
    if (res != writer.packet_size())
        std::println("Did not send entire packet");

    while (true) {
        int transferred = 0;
        res = libusb_interrupt_transfer(dev, BT_USB_DEV_EVT_ENDPOINT_NO, buffer, 257, &transferred, EVT_TIMEOUT);
        if (res == LIBUSB_ERROR_TIMEOUT)
            continue;
        if (res < 0)
            print_err_and_exit(res);

        std::println("Received {} bytes: {::02x}", transferred, std::span(buffer, transferred));

        auto evt = hci_evt_reader(buffer);
        if (evt.packet_size() != transferred) {
            std::println("Packet size mismatch: received {}, but packet reports {}", transferred, evt.packet_size());
            break;
        }
        switch (evt.get_code()) {
            case HCI_EVT_CMD_STATUS: {
                const auto err = evt.read_u8(0x00);
                auto opcode = evt.read_u16le(0x02);
                std::println("Received command {:#04x} status: {}", opcode, err);
                continue;
            }
            case HCI_EVT_REMOTE_NAME: {
                std::println("Received remote name event");
                break;
            }
            default:
                std::println("Unhandled event: {:#02x}", (uint8_t)evt.get_code());
        }
    }

    return 0;
}
```

Result:
```
Received 6 bytes: [0f, 04, 00, 02, 19, 04]
Received command 0x419 status: 0
Received 49 bytes: [00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00]
Packet size mismatch: received 49, but packet reports 2
```

However, when the timeout was large

```cpp
#define EVT_TIMEOUT 3000
```

The result was
```
Received 6 bytes: [0f, 04, 00, 02, 19, 04]
Received command 0x419 status: 0
Received 257 bytes: [07, ....] // (truncated for privacy)
Received remote name event
```

What I hadn't realised was that partial data could be returned even with a `LIBUSB_ERROR_TIMEOUT` and the final part of the message would return `LIBUSB_SUCCESS`. 

What makes this worse is that I had already read the relevant section of the [libusb docs](https://libusb.sourceforge.io/api-1.0/group__libusb__syncio.html) several times over, somehow not understanding that it'd behave that way.
