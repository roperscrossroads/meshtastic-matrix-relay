# Simplex-Chat Bridge Feasibility Study

## Executive Summary

This document evaluates the feasibility of modifying the Meshtastic-Matrix-Relay (MMRelay) project to bridge Meshtastic mesh networks with Simplex-Chat. After thorough analysis of both systems, **this integration is technically feasible** but would require significant architectural changes and faces several challenges.

**Feasibility Rating: MODERATE (60% feasible with significant effort)**

## Table of Contents

1. [Understanding the Current Architecture](#understanding-the-current-architecture)
2. [Simplex-Chat Overview](#simplex-chat-overview)
3. [Technical Feasibility Analysis](#technical-feasibility-analysis)
4. [Required Changes](#required-changes)
5. [Challenges and Limitations](#challenges-and-limitations)
6. [Implementation Approaches](#implementation-approaches)
7. [Recommendations](#recommendations)
8. [Resources](#resources)

---

## Understanding the Current Architecture

### Current MMRelay Architecture

MMRelay is designed as a **bidirectional bridge** between Meshtastic mesh networks and Matrix chat rooms. Key architectural components:

#### 1. **Dual Protocol Handler**
- **Meshtastic Side**: Uses `meshtastic-python` library
  - Supports serial, network (TCP), BLE, and MQTT connections
  - Event-driven message reception via pubsub callbacks
  - Synchronous connection with asyncio threading
  
- **Matrix Side**: Uses `matrix-nio` library
  - AsyncIO-based client with full E2EE support
  - Event callbacks for room messages, reactions, replies
  - Persistent session storage

#### 2. **Core Message Flow**

```
Meshtastic Device → on_meshtastic_message() → matrix_relay() → Matrix Room
Matrix Room → on_room_message() → send_reply_to_meshtastic() → Meshtastic Device
```

#### 3. **Key Features**
- **Bidirectional message relay** with formatting
- **Reactions and replies** support (with message ID mapping)
- **Multi-meshnet** support with custom prefixes
- **Message queue** to respect Meshtastic rate limits
- **SQLite database** for node info and message mapping
- **Plugin system** for extensibility
- **Channel mapping**: 1:1 mapping between Meshtastic channels and Matrix rooms

#### 4. **Critical Dependencies**
- Configuration via YAML
- Persistent credentials (Matrix access tokens)
- Message map storage for reactions/replies
- Node name resolution and caching

---

## Simplex-Chat Overview

### What is Simplex-Chat?

Simplex-Chat is a **privacy-focused messaging platform** with the following characteristics:

#### Core Features
1. **No user identifiers** - Uses pairwise queue-based connections
2. **Decentralized** - Self-hostable SMP (Simplex Messaging Protocol) servers
3. **E2EE by default** - Double-ratchet encryption
4. **Multiple clients** - CLI, Desktop, Mobile apps
5. **Group chats** - Via invitation links
6. **File transfers** - Encrypted file sharing

#### API/Integration Options

Simplex-Chat provides several integration methods:

1. **simplex-chat CLI** (Haskell-based command-line client)
   - Terminal interface with REPL
   - JSON output for automation
   - Command-based interaction
   - Local database storage
   
2. **Python API** (Third-party, community-maintained)
   - `simplex-chat-py` - Wrapper around CLI
   - Limited, not officially supported
   - Status: Experimental
   
3. **Chat Bots API** (Official, new in v5.8+)
   - HTTP API for bot integration
   - JSON-based request/response
   - Webhook support
   - Event-driven architecture
   - Documentation: https://github.com/simplex-chat/simplex-chat/blob/stable/docs/CHAT-BOTS.md

4. **Core Haskell Library**
   - `simplex-chat` library
   - Would require Haskell bindings
   - Not practical for Python integration

### Simplex Architecture Differences from Matrix

| Aspect | Matrix | Simplex-Chat |
|--------|--------|--------------|
| **Identifiers** | User IDs (@user:server) | No persistent IDs, connection-based |
| **Rooms/Groups** | Room IDs (!room:server) | Group links, no global IDs |
| **Federation** | Server-to-server federation | No federation, P2P connections |
| **Client Libraries** | Mature (matrix-nio, etc.) | Limited (CLI-based, new Bot API) |
| **Message IDs** | Event IDs (persistent) | Local message IDs (not globally unique) |
| **Reactions/Replies** | Native protocol support | Limited support, chat-level |
| **Discovery** | Room directory, aliases | Invitation links only |

---

## Technical Feasibility Analysis

### ✅ What Would Work

1. **Basic Message Relay**
   - Simplex → Meshtastic: Receive messages via Bot API/CLI, relay to mesh
   - Meshtastic → Simplex: Receive mesh messages, send via Bot API/CLI
   - **Feasibility: HIGH** - Straightforward bidirectional relay

2. **Group Chat Support**
   - Map Meshtastic channels to Simplex groups
   - One group per Meshtastic channel
   - **Feasibility: MODERATE** - Requires group management

3. **User Identity Mapping**
   - Map Meshtastic node IDs/names to Simplex conversation participants
   - Store mapping in SQLite database (similar to current node name storage)
   - **Feasibility: HIGH** - Similar to existing node tracking

4. **Plugin System**
   - Plugins could work with Simplex messages
   - Would need new plugin interface for Simplex events
   - **Feasibility: MODERATE** - Requires plugin API extension

### ⚠️ What Would Be Challenging

1. **Message ID Mapping for Reactions/Replies**
   - Simplex message IDs are local to each client, not globally unique
   - Current system relies on Matrix event IDs and Meshtastic message IDs
   - **Challenge Level: HIGH** - Would require custom message tracking
   - **Workaround**: Use message content hashing or timestamps

2. **Asyncio Integration**
   - Simplex CLI is synchronous and interactive
   - Bot API is HTTP-based but requires polling or webhooks
   - Current code is async-first (Matrix nio is async)
   - **Challenge Level: MODERATE** - Need async wrappers

3. **Connection Management**
   - Simplex connections are invitation-based, not room-based
   - No persistent room IDs like Matrix
   - Would need to manage connection state differently
   - **Challenge Level: MODERATE-HIGH** - Architectural change needed

4. **Reactions and Replies**
   - Simplex has limited native support for reactions/replies
   - Would need to emulate with text formatting or use new API features
   - **Challenge Level: HIGH** - Feature parity difficult

5. **Multi-Meshnet Support**
   - Current system uses Matrix custom fields for meshnet identification
   - Simplex doesn't support custom message fields easily
   - **Challenge Level: MODERATE** - Could use text prefixes

### ❌ What Would Not Work Well

1. **E2EE with Multiple Participants**
   - Simplex E2EE is pairwise between clients
   - Matrix E2EE allows room-level encryption with multiple users
   - Bridging E2EE content breaks end-to-end property
   - **Limitation: Fundamental** - E2EE would be bridge-terminated

2. **Dynamic Room Discovery**
   - Matrix has room directories and aliases
   - Simplex relies on invitation links
   - No automated discovery mechanism
   - **Limitation: Architectural** - Manual setup required

3. **Presence/Typing Indicators**
   - Matrix supports presence (online/offline) and typing indicators
   - Simplex intentionally doesn't expose this for privacy
   - **Limitation: By design** - Privacy-focused choice

4. **Message Editing/Deletion**
   - Matrix supports full message editing and redaction
   - Simplex has limited edit/delete support
   - **Limitation: Protocol** - Feature mismatch

---

## Required Changes

To implement a Simplex-Chat bridge, the following changes would be required:

### 1. **New Module: `simplex_utils.py`**

Create a new utility module parallel to `matrix_utils.py`:

```python
# src/mmrelay/simplex_utils.py

"""
Simplex-Chat integration utilities for MMRelay.

This module provides functions to connect to Simplex-Chat via the Bot API
or CLI interface, and relay messages bidirectionally with Meshtastic.
"""

import asyncio
import json
import subprocess
from typing import Optional, Dict, List

from mmrelay.log_utils import get_logger

logger = get_logger(name="Simplex")

# Global Simplex client reference
simplex_client = None
simplex_groups: List[dict] = []
config = None

async def connect_simplex(passed_config=None):
    """
    Connect to Simplex-Chat via Bot API or CLI.
    """
    # Implementation would go here
    pass

async def simplex_relay(group_id, message, longname, meshnet_name):
    """
    Relay a message to Simplex-Chat.
    """
    # Implementation would go here
    pass

async def on_simplex_message(message_data):
    """
    Handle incoming Simplex messages and relay to Meshtastic.
    """
    # Implementation would go here
    pass
```

### 2. **Configuration Schema Extension**

Add Simplex configuration to `config.yaml`:

```yaml
simplex:
  # Connection method: 'bot_api' or 'cli'
  connection_type: bot_api
  
  # For Bot API (recommended)
  bot_api:
    api_url: http://localhost:5224
    api_token: "your-bot-token"
    webhook_url: null  # Optional webhook for push notifications
  
  # For CLI (alternative)
  cli:
    executable_path: /usr/local/bin/simplex-chat
    profile: default
  
  # Group mappings
  groups:
    - id: "group-link-or-id"
      name: "Mesh Group 1"
      meshtastic_channel: 0

# Updated matrix_rooms to support mode selection
bridge_mode: matrix  # Options: 'matrix', 'simplex', 'both'

matrix_rooms:
  # ... existing Matrix config

simplex_groups:
  - id: "simplex-group-link"
    name: "Test Group"
    meshtastic_channel: 0
```

### 3. **Main Loop Modifications**

Update `main.py` to handle Simplex connections:

```python
async def main(config):
    # ... existing setup ...
    
    bridge_mode = config.get('bridge_mode', 'matrix')
    
    if bridge_mode in ['matrix', 'both']:
        # Connect to Matrix (existing code)
        matrix_client = await connect_matrix(passed_config=config)
        # ... existing Matrix setup ...
    
    if bridge_mode in ['simplex', 'both']:
        # Connect to Simplex
        from mmrelay.simplex_utils import connect_simplex, start_simplex_listener
        simplex_client = await connect_simplex(passed_config=config)
        
        # Start Simplex message listener
        asyncio.create_task(start_simplex_listener())
    
    # ... rest of main loop ...
```

### 4. **Database Schema Extension**

Add tables for Simplex-specific data:

```sql
-- Simplex group to Meshtastic channel mapping
CREATE TABLE IF NOT EXISTS simplex_groups (
    group_id TEXT PRIMARY KEY,
    name TEXT,
    meshtastic_channel INTEGER
);

-- Simplex contact mapping
CREATE TABLE IF NOT EXISTS simplex_contacts (
    contact_id TEXT PRIMARY KEY,
    display_name TEXT,
    meshtastic_id TEXT
);

-- Simplex message mapping (for reactions/replies)
CREATE TABLE IF NOT EXISTS simplex_message_map (
    simplex_msg_id TEXT PRIMARY KEY,
    meshtastic_id INTEGER,
    group_id TEXT,
    timestamp INTEGER,
    text TEXT
);
```

### 5. **Plugin System Extension**

Extend the plugin base class to support Simplex:

```python
# src/mmrelay/plugins/base_plugin.py

@abstractmethod
async def handle_simplex_message(self, message_data, formatted_message, contact_name, group_id):
    """
    Handle incoming Simplex messages.
    
    Args:
        message_data: Raw Simplex message data
        formatted_message: Formatted message text
        contact_name: Display name of sender
        group_id: Simplex group identifier
    
    Returns:
        bool: True if plugin handled the message, False otherwise
    """
    pass
```

### 6. **Message Queue Adaptation**

The existing message queue would work with minor modifications to support Simplex send operations.

---

## Challenges and Limitations

### Technical Challenges

1. **Simplex Bot API Maturity**
   - Bot API is relatively new (v5.8+)
   - Limited documentation and examples
   - May have bugs or missing features
   - Community support is smaller than Matrix

2. **Lack of Python Client Library**
   - No official Python library
   - Would need to build HTTP API wrapper or use CLI
   - CLI approach is less reliable (parsing output)
   - HTTP API approach requires polling or webhook server

3. **Message ID Management**
   - Simplex doesn't have global message IDs
   - Makes reaction/reply mapping complex
   - Would need custom message tracking
   - Could use content hashing as fallback

4. **Group Management Complexity**
   - Simplex groups use invitation links
   - No persistent, predictable group IDs
   - Requires manual setup for each group
   - Group membership changes need monitoring

5. **Testing and Development**
   - Requires running Simplex server locally
   - Bot API setup is non-trivial
   - Limited testing frameworks available
   - Harder to create automated tests

### Operational Challenges

1. **Configuration Complexity**
   - Users need to understand both Matrix and Simplex
   - Group invitation links need manual distribution
   - No automated room/group discovery
   - More moving parts = more potential issues

2. **Maintenance Burden**
   - Two separate protocols to maintain
   - Different update cycles and breaking changes
   - Double the debugging complexity
   - Simplex API could change significantly

3. **User Experience**
   - Simplex's privacy features (no IDs) conflict with bridging needs
   - Users expect consistent identities across bridge
   - Invitation-based model is less intuitive than Matrix rooms
   - Error messages and troubleshooting more complex

4. **Documentation Requirements**
   - Need to document Simplex setup
   - Explain differences from Matrix operation
   - Provide troubleshooting for both protocols
   - Update all deployment guides

---

## Implementation Approaches

### Approach 1: Bot API Integration (RECOMMENDED)

**Description**: Use the official Simplex Chat Bots API (v5.8+) with HTTP requests.

**Pros**:
- Official API, likely to be maintained
- HTTP-based, easier to integrate with Python
- Webhook support for real-time messages
- More reliable than parsing CLI output

**Cons**:
- Requires Simplex server v5.8+
- Needs webhook server for push notifications
- API is new and may have limitations
- Additional network service to run

**Implementation Outline**:
```python
import aiohttp

class SimplexBotClient:
    def __init__(self, api_url, api_token):
        self.api_url = api_url
        self.api_token = api_token
        self.session = None
    
    async def send_message(self, group_id, text):
        async with self.session.post(
            f"{self.api_url}/send",
            headers={"Authorization": f"Bearer {self.api_token}"},
            json={"groupId": group_id, "text": text}
        ) as resp:
            return await resp.json()
    
    async def poll_messages(self):
        # Poll for new messages
        async with self.session.get(
            f"{self.api_url}/messages",
            headers={"Authorization": f"Bearer {self.api_token}"}
        ) as resp:
            return await resp.json()
```

**Effort Estimate**: 3-4 weeks for experienced developer

### Approach 2: CLI Wrapper Integration

**Description**: Wrap the Simplex CLI tool with subprocess calls.

**Pros**:
- Works with any Simplex version that has CLI
- No additional server setup needed
- Can use all CLI features
- Simpler initial setup

**Cons**:
- Fragile (parsing CLI output)
- Performance overhead (process spawning)
- Harder to make async
- CLI changes break integration
- Less reliable message delivery

**Implementation Outline**:
```python
import asyncio
import json

class SimplexCLIClient:
    def __init__(self, cli_path):
        self.cli_path = cli_path
        self.process = None
    
    async def send_message(self, group_id, text):
        cmd = [self.cli_path, "send", group_id, text]
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        stdout, stderr = await proc.communicate()
        # Parse output...
    
    async def start_listener(self):
        # Start CLI in interactive mode and monitor output
        self.process = await asyncio.create_subprocess_exec(
            self.cli_path,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        # Monitor stdout for messages...
```

**Effort Estimate**: 2-3 weeks for experienced developer

### Approach 3: Hybrid Approach

**Description**: Support both Bot API and CLI, allowing users to choose.

**Pros**:
- Maximum compatibility
- Users can choose based on their setup
- Fallback options if one method fails
- Better for different use cases

**Cons**:
- Most complex to implement
- Double the code to maintain
- More configuration options
- Increased testing requirements

**Effort Estimate**: 5-6 weeks for experienced developer

### Approach 4: Plugin-Based Bridge

**Description**: Implement Simplex as a plugin rather than core feature.

**Pros**:
- Minimal changes to core codebase
- Optional feature, doesn't affect Matrix-only users
- Easier to maintain separately
- Can be developed by community

**Cons**:
- Plugin system may need extension
- Limited access to core functionality
- More complex for users to set up
- May not integrate as cleanly

**Effort Estimate**: 2-3 weeks for experienced developer + plugin system enhancements

---

## Recommendations

### Short-Term Recommendation: PROTOTYPE WITH BOT API

**Rationale**:
1. Start with a **proof-of-concept plugin** using the Bot API (Approach 4)
2. Validate the integration works before committing to major changes
3. Gather user feedback on whether there's actual demand
4. Test Bot API reliability and feature completeness

**Steps**:
1. Create `simplex_bridge_plugin.py` as a community plugin
2. Implement basic message relay (Meshtastic ↔ Simplex)
3. Use Bot API for Simplex communication
4. Store minimal state in plugin database
5. Document setup and limitations

**Timeline**: 1-2 weeks for basic prototype

### Medium-Term Recommendation: CORE INTEGRATION IF VALIDATED

If the prototype proves successful and there's user demand:

1. **Implement Bot API integration** (Approach 1) as core feature
2. **Add configuration schema** for Simplex groups
3. **Extend database** with Simplex-specific tables
4. **Update documentation** comprehensively
5. **Create automated tests**

**Timeline**: 4-6 weeks for production-ready implementation

### Long-Term Recommendation: MODULAR BRIDGE ARCHITECTURE

Consider refactoring to support multiple chat platforms:

```
MMRelay Core (Meshtastic + Message Queue + Database)
    ├── Matrix Bridge Module
    ├── Simplex Bridge Module
    └── [Future: Discord, Telegram, XMPP, etc.]
```

This would make it easier to add other platforms in the future.

**Timeline**: 8-12 weeks for major refactoring

### What NOT to Do

❌ **Don't try to maintain feature parity** with Matrix integration
   - Simplex is fundamentally different
   - Focus on core messaging, skip advanced features initially
   - Reactions/replies can be text-based, not protocol-level

❌ **Don't use CLI wrapper as primary method**
   - Too fragile for production use
   - Bot API is the right approach

❌ **Don't integrate into core immediately**
   - Start with plugin to validate approach
   - Avoid disrupting existing Matrix users

---

## Resources

### Simplex-Chat Documentation
- **Official Repo**: https://github.com/simplex-chat/simplex-chat
- **Bot API Docs**: https://github.com/simplex-chat/simplex-chat/blob/stable/docs/CHAT-BOTS.md
- **Protocol Spec**: https://github.com/simplex-chat/simplexmq/blob/master/protocol/simplex-messaging.md
- **Website**: https://simplex.chat/

### Existing Integrations
- **Matrix Bridge (Appservice)**: https://github.com/simplex-chat/simplex-chat-matrix-bridge (proof that bridging is possible)
- **Python CLI Wrapper**: https://github.com/kragen/simplex-api (unofficial, experimental)
- **Go Bot Example**: https://github.com/simplex-chat/simplex-chat/tree/stable/packages/simplex-bot-example

### Meshtastic Resources
- **Python API**: https://meshtastic.org/docs/software/python/cli/
- **Protocol Buffers**: https://buf.build/meshtastic/protobufs

### Similar Projects
- **matterbridge**: Multi-protocol chat bridge (Go)
  - https://github.com/42wim/matterbridge
  - Shows patterns for bridging multiple protocols
  
- **matrix-appservice-bridge**: Matrix bridge framework
  - https://github.com/matrix-org/matrix-appservice-bridge
  - Shows how Matrix.org approaches bridging

---

## Conclusion

**Is it feasible?** YES, with caveats.

**Should it be done?** DEPENDS on:
1. **User demand** - Is there real interest in Simplex integration?
2. **Maintenance capacity** - Can the project support two chat protocols?
3. **Bot API stability** - Will Simplex Bot API remain stable?

**Recommended Path Forward**:
1. ✅ **Start with proof-of-concept plugin** (2-3 weeks)
2. ✅ **Gather community feedback** (1-2 weeks)
3. ⚠️ **Evaluate Bot API maturity** (ongoing)
4. 🔄 **Decide on core integration** (after validation)

**Risk Level**: MODERATE-HIGH
- Technical risk: Moderate (Bot API is new)
- Maintenance risk: High (two protocols)
- User experience risk: Moderate (Simplex is less intuitive)

**Overall Recommendation**: 
⭐ **Proceed with a LIMITED proof-of-concept plugin** to validate interest and technical viability before committing to a full integration. If successful, consider making it a core feature in a future major version (e.g., v2.0).

---

## Appendix A: Comparison Matrix

| Feature | Current (Matrix) | Simplex Integration | Notes |
|---------|------------------|---------------------|-------|
| Message Relay | ✅ Full | ✅ Full | Basic feature, should work well |
| Reactions | ✅ Native | ⚠️ Text-based | Simplex has limited protocol support |
| Replies | ✅ Native | ⚠️ Text-based | Similar to reactions |
| User IDs | ✅ Persistent | ❌ Connection-based | Fundamental difference |
| Group Discovery | ✅ Directory | ❌ Invitation only | Manual setup required |
| E2EE | ✅ Room-level | ⚠️ Bridge-terminated | Privacy implications |
| Multi-meshnet | ✅ Custom fields | ⚠️ Text prefixes | Workaround needed |
| Async Support | ✅ Native | ⚠️ Requires wrapper | Bot API is HTTP-based |
| Plugins | ✅ Full support | ⚠️ Needs extension | Plugin API needs update |
| Presence | ✅ Supported | ❌ Not available | By design in Simplex |
| File Transfer | ✅ Supported | ⚠️ Possible | Would need implementation |
| Admin Tools | ✅ Available | ⚠️ Limited | Less mature ecosystem |

Legend:
- ✅ Fully supported
- ⚠️ Partial/workaround needed
- ❌ Not possible/not supported

---

## Appendix B: Estimated Development Effort

### Phase 1: Proof of Concept (Plugin)
- **Duration**: 2-3 weeks
- **Effort**: 40-60 hours
- **Deliverables**:
  - Basic Simplex plugin
  - Bot API integration
  - Simple message relay
  - Setup documentation

### Phase 2: Core Integration
- **Duration**: 4-6 weeks
- **Effort**: 80-120 hours
- **Deliverables**:
  - Core Simplex utilities module
  - Configuration schema
  - Database extensions
  - Full bidirectional relay
  - Comprehensive tests
  - User documentation

### Phase 3: Feature Parity
- **Duration**: 4-6 weeks
- **Effort**: 80-120 hours
- **Deliverables**:
  - Reactions/replies (text-based)
  - Multi-meshnet support
  - Plugin system integration
  - Advanced configuration
  - Migration guide

### Total Estimated Effort: 200-300 hours (5-7 weeks full-time)

---

## Appendix C: Code Example - Minimal Plugin Prototype

Here's a minimal example of what a Simplex bridge plugin might look like:

```python
# custom-plugins/simplex_bridge_plugin.py

"""
Simplex-Chat bridge plugin for MMRelay.

This plugin enables bidirectional message relay between Meshtastic
and Simplex-Chat using the Simplex Bot API.
"""

import asyncio
import aiohttp
from mmrelay.plugins.base_plugin import BasePlugin

class SimplexBridgePlugin(BasePlugin):
    plugin_name = "simplex_bridge"
    
    def __init__(self):
        super().__init__()
        self.api_url = self.config.get("api_url", "http://localhost:5224")
        self.api_token = self.config.get("api_token")
        self.group_id = self.config.get("group_id")
        self.session = None
    
    def start(self):
        super().start()
        # Start background listener for Simplex messages
        asyncio.create_task(self._listen_simplex())
    
    async def handle_meshtastic_message(self, packet, formatted_message, 
                                       longname, meshnet_name):
        """Relay Meshtastic messages to Simplex."""
        if not formatted_message:
            return False
        
        # Format message for Simplex
        simplex_message = f"[{meshnet_name}] {longname}: {formatted_message}"
        
        # Send to Simplex group
        await self._send_simplex_message(simplex_message)
        
        return False  # Let other plugins process too
    
    async def handle_room_message(self, room, event, full_message):
        """Not used for Simplex bridge."""
        return False
    
    async def _send_simplex_message(self, text):
        """Send message to Simplex via Bot API."""
        if not self.session:
            self.session = aiohttp.ClientSession()
        
        try:
            async with self.session.post(
                f"{self.api_url}/api/v1/send",
                headers={"Authorization": f"Bearer {self.api_token}"},
                json={"groupId": self.group_id, "text": text}
            ) as resp:
                if resp.status != 200:
                    self.logger.error(f"Failed to send to Simplex: {resp.status}")
                    return False
                return True
        except Exception as e:
            self.logger.error(f"Error sending to Simplex: {e}")
            return False
    
    async def _listen_simplex(self):
        """Background task to poll Simplex for messages."""
        while True:
            try:
                messages = await self._poll_simplex_messages()
                for msg in messages:
                    await self._handle_simplex_message(msg)
            except Exception as e:
                self.logger.error(f"Error polling Simplex: {e}")
            
            await asyncio.sleep(2)  # Poll every 2 seconds
    
    async def _poll_simplex_messages(self):
        """Poll Simplex Bot API for new messages."""
        if not self.session:
            self.session = aiohttp.ClientSession()
        
        try:
            async with self.session.get(
                f"{self.api_url}/api/v1/messages",
                headers={"Authorization": f"Bearer {self.api_token}"}
            ) as resp:
                if resp.status == 200:
                    return await resp.json()
                return []
        except Exception as e:
            self.logger.error(f"Error polling Simplex: {e}")
            return []
    
    async def _handle_simplex_message(self, message_data):
        """Handle incoming Simplex message and relay to Meshtastic."""
        sender = message_data.get("sender", "Unknown")
        text = message_data.get("text", "")
        
        if not text:
            return
        
        # Format for Meshtastic
        mesh_message = f"[Simplex] {sender}: {text}"
        
        # Send to Meshtastic
        self.send_message(mesh_message, channel=0)
```

**Configuration Example:**

```yaml
custom-plugins:
  simplex_bridge:
    active: true
    api_url: http://localhost:5224
    api_token: your-bot-api-token
    group_id: your-group-id
    channels: [0]
```

This example demonstrates the basic structure needed for a Simplex bridge plugin and shows that the integration is technically feasible with reasonable effort.

---

*Document Version: 1.0*  
*Date: 2024*  
*Author: Feasibility Study - Simplex-Chat Integration*
