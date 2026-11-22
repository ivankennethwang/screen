# Screens

A free, open-source app for iPhone and iPad to see and control a Mac over
macOS Screen Sharing (VNC) on your local network.

**Status:** planning. See [docs/PLAN.md](docs/PLAN.md).

## Why

macOS has Screen Sharing built in, but Apple ships a client only for the Mac.
Good commercial iOS clients exist; this one is free and MIT-licensed.

## How it connects

- Speaks VNC (RFB), which the Mac already serves on port 5900.
- Signs in with your macOS username and password (Apple Remote Desktop
  authentication), or a VNC password if you set one.
- Finds Macs on the network through Bonjour.
- Away from home, connect through a VPN such as Tailscale, WireGuard or
  OpenVPN and use the Mac's VPN address.

Only the login is encrypted; the session itself is plain VNC. Use a VPN on any
network you don't trust.

## Mac setup

System Settings → General → Sharing → Screen Sharing: on, and allow access for
your user.

## Not affiliated

An independent project, not affiliated with Edovia or Apple. "Screens" is a
working name.

## License

[MIT](LICENSE)
