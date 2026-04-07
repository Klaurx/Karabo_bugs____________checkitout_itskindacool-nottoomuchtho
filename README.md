### Karabo GuiServer Bugs

(Reported to XFEL) Two issues found in the Karabo control framework GuiServer, affecting versions through 3.0.X-hotfix.
Both issues relate to how the server handles client session state and constructs outbound authorization requests.

### Contents
KGS01 Authorization state desynchronization via login-over-login(high)


KGS03 Unsafe JSON construction in authorization request(Low)

### Scope
Target: European XFEL Karabo Control Framework

Components: GuiServerDevice.cc, UserAuthClient.cc

Versions: through 3.0.X-hotfix
