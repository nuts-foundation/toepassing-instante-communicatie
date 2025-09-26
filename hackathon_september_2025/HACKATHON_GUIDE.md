# OZO FHIR-Matrix Bridge Hackathon Guide

## 🎯 **The Challenge: Connecting Healthcare Chat Systems**

**Your Mission**: Connect your chat system to our healthcare Matrix rooms and prove that messages automatically become FHIR-compliant healthcare records.

### 🏥 **The Healthcare Problem**
Healthcare teams use different chat systems that can't talk to each other. Messages aren't stored as proper healthcare records. Patient care suffers from communication silos.

### 💡 **Our Solution: Matrix Federation + FHIR Bridge**
We've built a bridge that connects any chat system to healthcare-compliant FHIR records through Matrix federation:

```
Your Chat System → Matrix Federation → OZO Healthcare Homeserver → FHIR Bridge → Healthcare Records
```

**What you'll prove**: Send a message from your chat system, and watch it automatically become a proper FHIR Communication resource that references real healthcare workflows.

## 🌐 OZO Environment Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| **Matrix Homeserver** | `https://matrix.ozo.headease.nl` | Matrix messaging server |
| **FHIR Server** | `https://fhir.matrix.ozo.headease.nl/fhir` | FHIR R4 compliant API |
| **FHIR Bridge** | `https://fhir-bridge.matrix.ozo.headease.nl` | Bridge monitoring |
| **Element Web Client** | `https://element.matrix.ozo.headease.nl` | Web Matrix client |
| **MAS** | `https://auth.matrix.ozo.headease.nl` | auth server |
| **MAS** | `https://keycloak.ozo.headease.nl` | Register users with specific UZI / email |

### Health Check Endpoints
```bash
# Check if services are running
curl https://fhir-bridge.matrix.ozo.headease.nl/api/v1/queue/stats  # Bridge health via queue stats
curl https://fhir.matrix.ozo.headease.nl/fhir/metadata              # FHIR server metadata
```

## 🎯 **LIVE HACKATHON RESOURCES** (Ready Now!)

We have **working Matrix rooms** connected to our FHIR bridge that you can use immediately:

### 🚀 **Available Rooms**:
   - **CommunicationRequest/245**: `#communicationrequest/245:matrix.ozo.headease.nl` ✅ LIVE
   - **CareTeam/244**: `#careteam/244:matrix.ozo.headease.nl` ✅ LIVE

### 🌐 **Direct Element URLs** (Use these to join):
   - https://element.matrix.ozo.headease.nl/#/room/#communicationrequest/245:matrix.ozo.headease.nl
   - https://element.matrix.ozo.headease.nl/#/room/#careteam/244:matrix.ozo.headease.nl

### ✅ **Verify Resources Exist**:
```bash
# Check the working CommunicationRequest
curl "https://fhir.matrix.ozo.headease.nl/fhir/CommunicationRequest/245" -H "Accept: application/fhir+json"

# Check the working CareTeam
curl "https://fhir.matrix.ozo.headease.nl/fhir/CareTeam/244" -H "Accept: application/fhir+json"
```

### 🏗️ **Room Structure**
- **Format**: `#communicationrequest/{id}:matrix.ozo.headease.nl`
- Each room represents a FHIR CommunicationRequest (healthcare conversation thread)
- Messages in these rooms automatically create FHIR Communication resources

## 🧪 Main Challenge: Cross-Homeserver Healthcare Messaging

### 🎯 **Your Hackathon Challenge**

**Connect your chat system to our healthcare rooms** and prove cross-system healthcare messaging works:

```
Your Chat System → Matrix Federation → OZO Healthcare Homeserver → FHIR Bridge → Healthcare Records
```

**The Demo Flow**:
1. **We provide live healthcare Matrix rooms** (CommunicationRequest & CareTeam)
2. **You connect your chat system** to Matrix (Element, custom client, or bridge)
3. **Roland invites you cross-homeserver** to our healthcare rooms
4. **You send messages from your chat system**
5. **Matrix federation delivers your message** to our OZO healthcare homeserver
6. **Our FHIR bridge automatically converts messages** to FHIR Communication resources that reference real healthcare workflows
7. **Success**: Two different chat systems now talk through healthcare-compliant records! 🏥✨

## 🚀 **How to Complete the Challenge**

### **Step 1: Build Your Matrix Application Service/Bridge**
**Required**: Build a Matrix Application Service (AS) or bridge to connect your chat system:

- **Use Matrix SDK** (JavaScript, Python, Java, etc.) to build your bridge
- **Connect your existing chat system** to Matrix protocol through your AS/bridge
- **Register your AS** with your own homeserver or use Matrix.org
- **Matrix federation** will handle delivering messages to our healthcare homeserver

### **Step 2: Get Invited to Healthcare Rooms**
Roland will invite your Matrix user to our live healthcare rooms:
- `#communicationrequest/245:matrix.ozo.headease.nl` (Healthcare conversation thread)
- `#careteam/244:matrix.ozo.headease.nl` (Care team coordination)

### **Step 3: Send Your Healthcare Message**
Send a message from your chat system. It will flow through this path:

```
Your Chat → Matrix Federation → OZO Healthcare Homeserver → FHIR Bridge → Healthcare Record
```

Example message: *"Patient updates from [YOUR_TEAM]: Vitals stable, discharge planning initiated"*

### **Step 4: Prove It Worked - Check Healthcare Records**

Your message automatically becomes a healthcare record! Verify it exists in our FHIR server:

```bash
# Search for your message as a FHIR Communication resource
curl "https://fhir.matrix.ozo.headease.nl/fhir/Communication?part-of=CommunicationRequest/245" \
  -H "Accept: application/fhir+json"
```

**🎉 Success**: You'll see your chat message is now a proper healthcare Communication record that references CommunicationRequest/245!

**Expected FHIR Communication:**
```json
{
  "resourceType": "Bundle",
  "entry": [
    {
      "resource": {
        "resourceType": "Communication",
        "id": "generated-id",
        "status": "completed",
        "identifier": [
          {
            "system": "https://matrix.org/event-id",
            "value": "$matrix_event_id",
            "use": "official"
          }
        ],
        "partOf": [
          {
            "reference": "CommunicationRequest/245"
          }
        ],
        "sender": {
          "reference": "Patient/hackathon-patient"
        },
        "payload": [
          {
            "contentString": "Hello from hackathon team! Testing FHIR bridge integration 🚀"
          }
        ],
        "extension": [
          {
            "url": "http://hl7.org/fhir/StructureDefinition/communication-source-system",
            "valueString": "matrix"
          }
        ]
      }
    }
  ]
}
```

## 🏆 **Success Criteria: You've Connected Healthcare Chat Systems!**

**🎉 Hackathon Success When**:
1. ✅ **Chat System Connected** - Your chat system can send messages through Matrix
2. ✅ **Cross-Homeserver Federation Works** - Messages reach our healthcare homeserver
3. ✅ **FHIR Healthcare Records Created** - Your chat message becomes a proper FHIR Communication with:
   - Your message text in the healthcare record
   - Reference to CommunicationRequest/245 (real healthcare workflow)
   - Matrix event ID for traceability
   - Proper healthcare metadata and compliance
4. ✅ **Healthcare Interoperability Proven** - Different chat systems now communicate through healthcare-compliant records! 🏥

**You've just solved healthcare communication silos!**

## 🚀 **What Makes This Special?**

### **Why This Matters for Healthcare**
- **Compliance**: Messages automatically meet FHIR healthcare standards
- **Interoperability**: Any chat system can talk to any other through healthcare records
- **Audit Trail**: Every message has proper healthcare metadata and traceability
- **Federation**: No vendor lock-in - different organizations can use different systems
- **Real Healthcare Workflows**: Your messages reference actual CommunicationRequests and CareTeams

### **Technical Innovation**
- **Matrix Protocol**: Decentralized, encrypted, federated messaging
- **FHIR R4**: Global healthcare data exchange standard
- **Bidirectional Sync**: Changes in FHIR update Matrix, and vice versa
- **Cross-System Bridge**: Your custom chat ↔ Healthcare records seamlessly

---