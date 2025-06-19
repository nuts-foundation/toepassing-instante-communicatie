# Healthcare Communication Hackathon Guide
## Implementing FHIR-Matrix Bridge Integration

This guide helps hackathon participants understand how to implement their own healthcare communication bridge based on the Matrix protocol specification. Instead of using the existing `matrix-fhir-bridge`, you'll learn the core concepts to build your own implementation for any healthcare platform.

---

## 🎯 Goal

Build a secure communication bridge that connects your healthcare platform to Matrix-based instant messaging, enabling standardized care team coordination and patient communication.

---

## 📋 Prerequisites

- Understanding of your platform's data model (FHIR, HL7, proprietary)
- Basic knowledge of Matrix protocol
- REST API development experience
- Healthcare data standards awareness

---

## 🏗️ Implementation Overview

Your bridge needs to handle two main workflows:
1. **Data Synchronization**: Sync patients, practitioners, and related persons
2. **Message Exchange**: Enable bi-directional communication

---

## Phase 1: Data Synchronization Setup

### Step 1.1: Identity Mapping Strategy

**Core Concept**: Map your platform's user identities to Matrix user IDs using third-party identifiers (3PIDs).

**For Different User Types:**

**Healthcare Practitioners:**
```json
{
  "platform_id": "doc_12345",
  "matrix_id": "@dr.smith:hospital-a.nl",
  "threepids": [
    {
      "medium": "uzi_nr",
      "address": "123456789"
    },
    {
      "medium": "ura_nr",
      "address": "00000001"
    },
    {
      "medium": "role_code",
      "address": "158965000"
    },
    {
      "medium": "email",
      "address": "dr.smith@hospital-a.nl"
    }
  ]
}
```

**Patients & Related Persons:**
```json
{
  "platform_id": "patient_67890",
  "matrix_id": "@john.doe:platform.nl",
  "threepids": [
    {
      "medium": "email",
      "address": "john.doe@example.com"
    }
  ]
}
```

**Implementation Task:**
- Create a mapping table in your database
- Implement identity resolution service
- Handle edge cases (missing emails, duplicate identifiers)

### Step 1.2: Matrix Account Management

**Matrix User Creation Process:**

1. **Check Existing Account:**
   ```javascript
   try {
     const profile = await matrixClient.getProfileInfo("@user:domain.nl");
     console.log("User exists:", profile);
   } catch (error) {
     console.log("User does not exist, needs creation");
   }
   ```

2. **Create New Account (if needed):**
   ```javascript
   const registerResponse = await matrixClient.register(
     "dr.smith",
     "secure_generated_password",
     null, // session
     {
       initial_device_display_name: "Healthcare Bridge"
     }
   );
   ```

3. **Add 3PIDs for Identity Verification:**
   ```javascript
   // Add email 3PID
   await matrixClient.addThreePid({
     sid: "session_id",
     client_secret: "secret",
     auth: {
       type: "m.login.password",
       user: "@dr.smith:domain.nl",
       password: "secure_generated_password"
     }
   });
   ```

**Implementation Task:**
- Build user provisioning service
- Implement 3PID validation
- Handle Matrix homeserver assignment based on organization

### Step 1.3: Care Team Synchronization

**Concept**: Map your platform's care teams to Matrix Spaces.

**Data Mapping Pattern:**
```javascript
// Your platform's care team
const careTeam = {
  id: "team_123",
  patient_id: "patient_67890",
  members: [
    { type: "practitioner", id: "doc_12345", role: "primary" },
    { type: "practitioner", id: "nurse_456", role: "care_coordinator" },
    { type: "related_person", id: "family_789", role: "caregiver" }
  ]
};

// Convert to Matrix Space
const matrixSpace = {
  name: `Care Team for ${patient.name}`,
  alias: `#careteam-${careTeam.id}:domain.nl`,
  topic: "Healthcare coordination space",
  power_levels: {
    users: {
      "@dr.smith:domain.nl": 75,    // Practitioner
      "@nurse.jane:domain.nl": 75,  // Practitioner
      "@family.member:domain.nl": 50, // Related person
      "@patient.john:domain.nl": 25   // Patient
    }
  }
};
```

**Matrix Space Creation:**
```javascript
const spaceId = await matrixClient.createRoom({
  creation_content: {
    type: "m.space"
  },
  name: "Care Team for John Doe",
  room_alias_name: "careteam-123",
  power_level_content_override: {
    users: {
      "@dr.smith:domain.nl": 75,
      "@nurse.jane:domain.nl": 75,
      "@family.member:domain.nl": 50,
      "@patient.john:domain.nl": 25
    }
  }
});
```

**Implementation Tasks:**
- Monitor care team changes in your platform
- Create/update Matrix spaces automatically
- Manage member permissions based on roles
- Handle patient privacy (no patient data in room metadata)

### Step 1.4: Matrix Identity Server Bridge Implementation

**Core Concept**: Implement a bridge service that connects ma1sd (Matrix Identity Server) to your backend identity system (Keycloak, database, mCSD, etc.).

**Identity Server Bridge Architecture:**
```
Matrix Client → Synapse → ma1sd → Your Bridge Service → Backend System
                                 (localhost:3000)     (Keycloak/mCSD/DB)
```

**Bridge Service Interface:**
```javascript
class IdentityServerBridge {
  constructor(backendAdapter) {
    this.backend = backendAdapter; // Abstracted backend interface
    this.app = express();
    this.setupRoutes();
  }

  // Backend adapter interface - implement for your platform
  async searchUsersByAttribute(medium, address) {
    return this.backend.searchUsers(medium, address);
  }

  async authenticateUser(username, password) {
    return this.backend.authenticate(username, password);
  }

  async getUserProfile(username) {
    return this.backend.getProfile(username);
  }
}
```

**Required ma1sd REST API Endpoints:**

**1. Single Identity Lookup** - Convert 3PID to Matrix user ID:
```javascript
app.post('/_ma1sd/backend/api/v1/identity/single', async (req, res) => {
  try {
    const { medium, address } = req.body.lookup;

    // Search your backend for user with this 3PID
    const users = await this.backend.searchUsers(medium, address);

    if (users.length === 0) {
      return res.json({}); // No user found
    }

    const user = users[0];
    res.json({
      id: {
        value: this.formatMatrixId(user.username),
        type: 'localpart'
      },
      display_name: user.displayName || user.username
    });
  } catch (error) {
    console.error('Identity lookup error:', error);
    res.json({});
  }
});
```

**2. Bulk Identity Lookup** - Batch convert multiple 3PIDs:
```javascript
app.post('/_ma1sd/backend/api/v1/identity/bulk', async (req, res) => {
  try {
    const { lookup } = req.body; // Array of {medium, address}
    const results = [];

    for (const item of lookup) {
      const users = await this.backend.searchUsers(item.medium, item.address);
      if (users.length > 0) {
        const user = users[0];
        results.push({
          address: item.address,
          medium: item.medium,
          id: {
            value: this.formatMatrixId(user.username),
            type: 'localpart'
          },
          display_name: user.displayName || user.username
        });
      }
    }

    res.json({ results });
  } catch (error) {
    console.error('Bulk lookup error:', error);
    res.json({ results: [] });
  }
});
```

**3. Directory Search** - Find users by name:
```javascript
app.post('/_ma1sd/backend/api/v1/directory/user/search', async (req, res) => {
  try {
    const { search_term } = req.body;

    const users = await this.backend.searchUsersByName(search_term);

    const results = users.map(user => ({
      value: this.formatMatrixId(user.username),
      display_name: user.displayName || user.username
    }));

    res.json({
      limited: false,
      results
    });
  } catch (error) {
    console.error('Directory search error:', error);
    res.json({ limited: false, results: [] });
  }
});
```

**4. Authentication** - Validate user credentials:
```javascript
app.post('/_ma1sd/backend/api/v1/auth/login', async (req, res) => {
  try {
    const { mxid, password } = req.body.auth;
    const username = mxid.split(':')[0].substring(1); // Extract from @user:domain

    const authResult = await this.backend.authenticate(username, password);

    if (!authResult.success) {
      return res.json({ success: false });
    }

    const user = await this.backend.getProfile(username);

    res.json({
      success: true,
      id: {
        type: 'localpart',
        value: username
      },
      profile: {
        display_name: user.displayName,
        three_pids: this.extractThreePids(user),
        roles: user.roles || []
      }
    });
  } catch (error) {
    console.error('Auth error:', error);
    res.json({ success: false });
  }
});
```

**Backend Adapter Implementations:**

**For Keycloak Backend:**
```javascript
class KeycloakAdapter {
  constructor(config) {
    this.keycloakClient = axios.create({
      baseURL: config.keycloakUrl,
      timeout: 10000
    });
    this.clientId = config.clientId;
    this.clientSecret = config.clientSecret;
    this.realm = config.realm;
  }

  async searchUsers(medium, address) {
    const token = await this.getServiceToken();

    // Map 3PID medium to Keycloak attribute
    const attributeMap = {
      'email': 'email',
      'ura_nr': 'uraNr',
      'uzi_nr': 'uziNr',
      'role_code': 'roleCode'
    };

    const attribute = attributeMap[medium] || medium;

    if (medium === 'email') {
      const response = await this.keycloakClient.get(
        `/admin/realms/${this.realm}/users?email=${encodeURIComponent(address)}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      return response.data;
    }

    // Search custom attributes
    const response = await this.keycloakClient.get(
      `/admin/realms/${this.realm}/users?q=${attribute}:${encodeURIComponent(address)}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  }

  async authenticate(username, password) {
    try {
      await this.keycloakClient.post(
        `/realms/${this.realm}/protocol/openid-connect/token`,
        new URLSearchParams({
          grant_type: 'password',
          client_id: this.clientId,
          client_secret: this.clientSecret,
          username,
          password
        }),
        { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
      );
      return { success: true };
    } catch (error) {
      return { success: false };
    }
  }
}
```

**For FHIR mCSD Backend:**
```javascript
class FhirMcsdAdapter {
  constructor(config) {
    this.fhirClient = fhirKitClient({
      baseUrl: config.fhirBaseUrl,
      resourceType: 'Practitioner'
    });
  }

  async searchUsers(medium, address) {
    const searchParams = {};

    // Map 3PID medium to FHIR search parameters
    switch (medium) {
      case 'email':
        searchParams.telecom = `email|${address}`;
        break;
      case 'uzi_nr':
        searchParams.identifier = `http://fhir.nl/fhir/NamingSystem/uzi-nr-pers|${address}`;
        break;
      case 'ura_nr':
        // Search PractitionerRole for URA number
        const roles = await this.fhirClient.search({
          resourceType: 'PractitionerRole',
          searchParams: {
            organization: `urn:oid:2.16.528.1.1007.3.3|${address}`
          }
        });

        if (roles.entry && roles.entry.length > 0) {
          const practitionerRefs = roles.entry.map(e =>
            e.resource.practitioner?.reference?.replace('Practitioner/', '')
          ).filter(Boolean);

          if (practitionerRefs.length > 0) {
            searchParams._id = practitionerRefs.join(',');
          }
        }
        break;
    }

    const bundle = await this.fhirClient.search({
      resourceType: 'Practitioner',
      searchParams
    });

    return this.transformFhirUsers(bundle.entry || []);
  }

  transformFhirUsers(entries) {
    return entries.map(entry => {
      const practitioner = entry.resource;
      const matrixTelecom = practitioner.telecom?.find(t =>
        t.system === 'other' &&
        t.value?.includes('@') &&
        t.extension?.some(ext => ext.valueString === 'matrix-messaging')
      );

      return {
        id: practitioner.id,
        username: matrixTelecom ?
          matrixTelecom.value.split('@')[1].split(':')[0] :
          practitioner.id,
        displayName: practitioner.name?.[0] ?
          `${practitioner.name[0].given?.join(' ')} ${practitioner.name[0].family}` :
          practitioner.id,
        email: practitioner.telecom?.find(t => t.system === 'email')?.value,
        uziNr: practitioner.identifier?.find(i =>
          i.system === 'http://fhir.nl/fhir/NamingSystem/uzi-nr-pers'
        )?.value
      };
    });
  }
}
```

**For Database Backend:**
```javascript
class DatabaseAdapter {
  constructor(config) {
    this.db = config.database; // Your database connection
  }

  async searchUsers(medium, address) {
    const query = {
      email: 'SELECT * FROM users WHERE email = ?',
      ura_nr: 'SELECT * FROM users WHERE ura_number = ?',
      uzi_nr: 'SELECT * FROM users WHERE uzi_number = ?',
      role_code: 'SELECT * FROM users WHERE role_code = ?'
    };

    const sql = query[medium];
    if (!sql) return [];

    const results = await this.db.query(sql, [address]);
    return results.map(row => ({
      id: row.id,
      username: row.username,
      displayName: `${row.first_name} ${row.last_name}`,
      email: row.email,
      uziNr: row.uzi_number,
      uraNr: row.ura_number,
      roleCode: row.role_code
    }));
  }

  async authenticate(username, password) {
    const user = await this.db.query(
      'SELECT * FROM users WHERE username = ?',
      [username]
    );

    if (!user[0]) return { success: false };

    // Use your password hashing library
    const isValid = await bcrypt.compare(password, user[0].password_hash);
    return { success: isValid };
  }
}
```

**Configuration and Setup:**
```javascript
// Configure ma1sd.yaml to use your bridge
const ma1sdConfig = `
rest:
  enabled: true
  host: http://localhost:3000  # Your bridge service URL
`;

// Start your bridge service
const config = {
  // Keycloak config
  keycloakUrl: process.env.KEYCLOAK_URL,
  clientId: process.env.KEYCLOAK_CLIENT_ID,
  clientSecret: process.env.KEYCLOAK_CLIENT_SECRET,
  realm: process.env.KEYCLOAK_REALM,

  // Or FHIR config
  fhirBaseUrl: process.env.FHIR_BASE_URL,

  // Or database config
  database: yourDatabaseConnection
};

// Choose your backend adapter
const backendAdapter = new KeycloakAdapter(config);
// const backendAdapter = new FhirMcsdAdapter(config);
// const backendAdapter = new DatabaseAdapter(config);

const bridge = new IdentityServerBridge(backendAdapter);
bridge.start();
```

**Implementation Tasks:**
- Choose and implement the appropriate backend adapter for your platform
- Set up the bridge service with required ma1sd REST API endpoints
- Configure ma1sd to use your bridge service
- Test identity lookups and authentication flows
- Handle 3PID mapping for your platform's user identifiers

---

## Phase 2: Message Exchange Implementation

### Step 2.1: Conversation Rooms

**Concept**: Each conversation topic becomes a Matrix room within the care team space.

**Room Creation Pattern:**
```javascript
const roomId = await matrixClient.createRoom({
  name: "Medication Review Discussion",
  room_alias_name: "med-review-456",
  invite: [
    "@dr.smith:domain.nl",
    "@nurse.jane:domain.nl"
  ],
  initial_state: [
    {
      type: "m.space.parent",
      state_key: spaceId,
      content: {
        via: ["domain.nl"],
        canonical: true
      }
    }
  ]
});
```

**After Room Creation - Link to Space:**

After creating the room, you need to establish the bidirectional space-room relationship using state events:

```javascript
// 1. Add room as child to the space
await matrixClient.sendStateEvent(
  spaceId,                    // The care team space ID
  "m.space.child",           // EventType.SpaceChild
  roomId,                    // State key: the room ID
  {
    "via": ["domain.nl"],
    "order": "01",
    "suggested": true
  }
);

// 2. Add space as parent to the room
await matrixClient.sendStateEvent(
  roomId,                    // The conversation room ID
  "m.space.parent",         // EventType.SpaceParent
  spaceId,                  // State key: the space ID
  {
    "via": ["domain.nl"],
    "canonical": true
  }
);
```

### Step 2.2: User Identity Mapping in Rooms

**After creating rooms, you need to store user identity mappings as state events. This allows the bridge to track which platform users correspond to which Matrix users.**

**User Mapping Data Structure:**
```javascript
// Define the structure for user identity mapping
const userMap = {
  "@dr.smith:domain.nl": {
    fhirRef: "Practitioner/doc_12345",        // FHIR resource reference
    uraNr: "00000001",                        // URA number (practitioners)
    uziNr: "123456789",                       // UZI number (practitioners)
    roleCode: "158965000",                    // Role code (practitioners)
    email: "dr.smith@hospital-a.nl"           // Email address
  },
  "@patient.john:domain.nl": {
    fhirRef: "Patient/patient_67890",         // FHIR resource reference
    email: "john.doe@example.com"             // Email address (patients/related persons)
  },
  "@family.member:domain.nl": {
    fhirRef: "RelatedPerson/related_456",     // FHIR resource reference
    email: "family.member@example.com"        // Email address (patients/related persons)
  }
};
```

**Storing User Mappings as State Events:**
```javascript
// Store user-specific identity mappings in the room
// Each user gets their own state event with their Matrix ID as the state key
const CUSTOM_USER_MAP_STATE_KEY = "custom.user_mappings";

await Promise.all(
  Object.entries(userMap).map(([userId, userInfo]) => {
    // Create an intent for each user to send their own state event
    const userIntent = matrixClient.getIntent ?
      matrixClient.getIntent(userId) : // Application service pattern
      matrixClient; // Regular client

    return userIntent.sendStateEvent(
      roomId,                          // Room ID where to store the mapping
      CUSTOM_USER_MAP_STATE_KEY,       // State event type
      userId,                          // State key: the Matrix user ID
      userInfo                         // Content: the user's platform identity info
    );
  })
);
```

**Reading User Mappings from Room State:**
```javascript
// Retrieve user identity mappings from room state
const getUserMapping = async (roomId, userId) => {
  try {
    const stateEvent = await matrixClient.getStateEvent(
      roomId,
      CUSTOM_USER_MAP_STATE_KEY,
      userId
    );
    return stateEvent; // Returns the userInfo object
  } catch (error) {
    console.warn(`No user mapping found for ${userId} in room ${roomId}`);
    return null;
  }
};

// Example usage:
const userMapping = await getUserMapping(roomId, "@dr.smith:domain.nl");
if (userMapping) {
  console.log(`User FHIR reference: ${userMapping.fhirRef}`);
  console.log(`User UZI number: ${userMapping.uziNr}`);
}
```

**Updating User Mappings:**
```javascript
// Update user mapping when platform data changes
const updateUserMapping = async (roomId, userId, updatedUserInfo) => {
  const userIntent = matrixClient.getIntent ?
    matrixClient.getIntent(userId) :
    matrixClient;

  await userIntent.sendStateEvent(
    roomId,
    CUSTOM_USER_MAP_STATE_KEY,
    userId,
    updatedUserInfo
  );
};

// Example: Update practitioner's role
await updateUserMapping(roomId, "@dr.smith:domain.nl", {
  fhirRef: "Practitioner/doc_12345",
  uraNr: "00000001",
  uziNr: "123456789",
  roleCode: "223366009", // Updated role code
  email: "dr.smith@hospital-a.nl"
});
```

**Alternative: Using HTTP API Directly:**
```javascript
// If you need to use raw HTTP requests instead of SDK
const addChildToSpace = async (spaceId, roomId) => {
  await fetch(`${homeserverUrl}/_matrix/client/v3/rooms/${spaceId}/state/m.space.child/${roomId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      via: ["domain.nl"],
      order: "01",
      suggested: true
    })
  });
};
```

### Step 2.3: Message Routing

**Platform → Matrix Flow:**

1. **Listen for New Messages in Your Platform:**
   ```javascript
   // Webhook or polling mechanism
   function handleNewMessage(message) {
     const matrixRoom = findMatrixRoom(message.conversation_id);
     const senderMatrixId = getUserMatrixId(message.sender_id);

     sendMatrixMessage(matrixRoom, senderMatrixId, message.content);
   }
   ```

2. **Send to Matrix:**
   ```javascript
   await matrixClient.sendMessage(roomId, {
     msgtype: "m.text",
     body: "Patient shows improvement in mobility",
     formatted_body: "<p>Patient shows improvement in mobility</p>",
     format: "org.matrix.custom.html"
   });
   ```

**Matrix → Platform Flow:**

1. **Listen for Matrix Events:**
   ```javascript
   matrixClient.on("Room.timeline", (event, room) => {
     if (event.getType() === "m.room.message") {
       const platformConversationId = extractPlatformId(room);
       const senderInfo = getUserMapping(event.getSender());

       createPlatformMessage({
         conversation_id: platformConversationId,
         sender_id: senderInfo.platform_id,
         content: event.getContent().body
       });
     }
   });
   ```

### Step 2.4: User Invitation Flow

**Cross-Organization Invites:**

When inviting users from other healthcare organizations, include patient context:

```javascript
await matrixClient.invite(roomId, "@specialist:other-hospital.nl", {
  reason: "Consultation request for cardiac care",
  "nl-ta-chat.invite.context": {
    version: "1.0",
    "room.subject": {
      platform_type: "patient_record",
      identifier: "patient_67890",
      display_name: "J. Doe"
    }
  }
});
```

**Implementation Task:**
- Build invitation context mapping for your platform
- Handle cross-platform patient identification
- Respect privacy requirements

---

## 🔧 Technical Implementation Guidelines

### Data Model Adaptation

**If Your Platform Uses FHIR:**
- Map directly using resource references
- Use existing FHIR identifiers for 3PIDs
- Leverage CareTeam and CommunicationRequest resources

**If Your Platform Has Custom Data Model:**
```javascript
// Example mapping for proprietary system
const platformToMatrix = {
  users: {
    staff_member: "practitioner",
    family_contact: "related_person",
    client: "patient"
  },
  teams: {
    care_circle: "care_team_space",
    treatment_group: "specialized_space"
  },
  communications: {
    care_note: "room_message",
    urgent_alert: "room_message_with_notification"
  }
};
```

### API Integration Patterns

**Event-Driven Synchronization:**
```javascript
// Set up webhooks in your platform
app.post('/webhooks/care-team-updated', (req, res) => {
  const careTeam = req.body;
  updateMatrixSpace(careTeam);
  res.status(200).send('OK');
});

app.post('/webhooks/message-created', (req, res) => {
  const message = req.body;
  forwardToMatrix(message);
  res.status(200).send('OK');
});
```

**Polling Alternative (if webhooks unavailable):**
```javascript
setInterval(async () => {
  const recentChanges = await fetchRecentChanges();
  for (const change of recentChanges) {
    await syncToMatrix(change);
  }
}, 30000); // Every 30 seconds
```

### Security Considerations

**Access Token Management:**
- Use Application Service pattern for server-to-server communication
- Implement token rotation
- Store credentials securely

**Patient Privacy:**
- Never include patient identifiers in room names or topics
- Use patient context only in invitation events
- Implement audit logging for all access

---

## 📚 Resources

- [Matrix Specification](https://spec.matrix.org/)
- [Matrix Client-Server API](https://spec.matrix.org/latest/client-server-api/)
- [FHIR R4 Specification](https://hl7.org/fhir/R4/)
- [Matrix Application Service API](https://spec.matrix.org/latest/application-service-api/)

---

**Good luck with your implementation! Focus on solving real healthcare communication challenges while building a secure, interoperable solution.**
