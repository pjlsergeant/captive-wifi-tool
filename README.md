# Captive WiFi Portal Debugger

Dependency-free script for trying to diagnose and fix why you can't connect to that public hotspot.

* Save [captive-wifi-tool](https://raw.githubusercontent.com/pjlsergeant/captive-wifi-tool/refs/heads/main/captive-wifi-tool) locally
* `./captive-wifi-tool`

and follow the instructions.

# What it does

Tries to figure out why you aren't connected to the internet. If you have DNS servers that are set to something that isn't DHCP, it'll offer to temporarily set your DNS to the DHCP server's one, fire up the portal, and then restore afterwards.

Otherwise it tries to print as much helpful information from OS X as it can

# What _exactly_ it does

```mermaid
flowchart TD
    subgraph Startup
        A[Start] --> B{Stale state<br/>from crash?}
        B -->|Yes| C[Prompt: restore DNS?]
        C -->|Yes| D[Restore DNS]
        C -->|No| E[Keep state]
        B -->|No| F[Continue]
        D --> F
        E --> F
    end

    subgraph Network Discovery
        F --> G[Find default route<br/>interface + gateway]
        G --> H[Validate sudo]
        H -->|Failed| X1[Exit]
        H -->|OK| I[Get network service name]
        I --> J[Get DHCP lease<br/>IP + netmask]
        J --> K{Valid IP?}
        K -->|Self-assigned| X1
        K -->|No IP| X1
        K -->|Valid| L[Get DNS settings<br/>System / Manual / DHCP]
    end

    subgraph Portal Detection
        L --> M[Build DNS test list<br/>DHCP first, then System]
        M --> N[For each DNS server]
        N --> O[For each portal domain<br/>Apple / Google / Firefox]
        O --> P[Test DNS resolution]

        P --> Q{Resolved?}
        Q -->|No| R1[DNS_BLOCKED]
        Q -->|To local IP| R2[DNS_INTERCEPTION]
        Q -->|To real IP| S[Test HTTP response]

        S --> T{Response?}
        T -->|3xx redirect| R3[HTTP_REDIRECT]
        T -->|2xx correct body| R4[INTERNET_WORKING]
        T -->|2xx wrong body| R5[HTTP_INJECTION]
        T -->|Error| R6[HTTP_FAILED]

        R3 --> U[Extract portal URL]
        R5 --> U
        U --> V[Test portal host DNS]

        R1 --> W[Next test]
        R2 --> W
        R4 --> W
        R6 --> W
        V --> W
        W --> O
    end

    subgraph Analysis
        W --> Y{Internet<br/>working?}
        Y -->|Yes| Z1[Show success + exit]
        Y -->|No| Z2[Analyze DNS mismatch]

        Z2 --> AA{Conflicts<br/>found?}
        AA -->|No| AB{HTTP<br/>interception?}
        AB -->|Yes| AC[Open neverssl.com]
        AB -->|No| AD[Show diagnostics]

        AA -->|Yes| AE[Show remediation steps]
    end

    subgraph Portal Mode
        AE --> AF{Manual DNS<br/>or VPN?}
        AF -->|No| AD
        AF -->|Yes| AG[Offer Portal Mode]
        AG -->|Decline| AD
        AG -->|Accept| AH[Backup current DNS]
        AH --> AI[Set DNS to DHCP servers]
        AI --> AJ[Flush DNS cache]
        AJ --> AK[Wait for user<br/>to login to portal]
        AK --> AL[Restore original DNS]
        AL --> AM[Clear state file]
    end

    Z1 --> END[End]
    AC --> END
    AD --> END
    AM --> END
    X1 --> END

    style R4 fill:#90EE90
    style Z1 fill:#90EE90
    style R1 fill:#FFB6C1
    style R2 fill:#FFB6C1
    style R6 fill:#FFB6C1
    style X1 fill:#FFB6C1
    style R3 fill:#FFE4B5
    style R5 fill:#FFE4B5
```

## Detection Types

| Result | Meaning |
|--------|---------|
| `DNS_BLOCKED` | DNS query failed - server unreachable or blocking |
| `DNS_INTERCEPTION` | DNS resolved to local IP - portal hijacking DNS |
| `HTTP_REDIRECT` | Got 3xx redirect to portal login page |
| `HTTP_INJECTION` | Got 2xx but body contains JS/meta redirect to portal |
| `HTTP_FAILED` | HTTP request failed - network blocked |
| `INTERNET_WORKING` | Successfully reached test endpoint |

# Author

Peter Sergeant -- pete@clueball.com