# Ghost Witness Registry

The official registry of approved witnesses and TEE endpoints for Ghost Witness Protocol.

## Purpose

This registry serves as the source of truth for:
- **Approved witness addresses** - ETH addresses authorized to sign proofs
- **Active TEE endpoints** - Verified TEE servers that users can connect to
- **Current epoch** - Protocol version and upgrade coordination

## Registry Structure

```json
{
  "epoch": 0,                      // Current epoch (increments on protocol upgrades)
  "protocol_version": "1.3",       // Ghost Witness protocol version
  "updated_at": "2025-12-30...",  // Last registry update
  
  "witnesses": [
    {
      "address": "0x9366cEE5...",     // Ethereum address of witness
      "name": "Official Phala Witness", // Human-readable name
      "operator": "Phala Network",     // Who operates this witness
      "status": "active"               // active | maintenance | retired
    }
  ],
  
  "tees": [
    {
      "endpoint": "https://...",           // TEE server URL
      "witness_address": "0x9366cEE5...",  // Which witness this TEE uses
      "name": "Official Phala TEE",        // Human-readable name
      "region": "US",                      // Geographic region
      "status": "active",                  // active | maintenance | retired
      "limits": {
        "requests_per_hour": 100,          // Rate limit
        "max_file_size_mb": 100            // Max file size
      },
      "features": {
        "ecies_encryption": true,          // Supports encrypted requests
        "session_cookies": true            // Supports authenticated downloads
      }
    }
  ]
}
```

## How Applications Use This Registry

### 1. Extension Fetches Registry on Startup

```javascript
const REGISTRY_URL = 'https://raw.githubusercontent.com/rachdiamine/templates/main/witness-registry.json';

async function loadRegistry() {
  const response = await fetch(REGISTRY_URL);
  const registry = await response.json();
  
  // Cache locally
  await chrome.storage.local.set({ registry });
  
  return registry;
}
```

### 2. User Selects TEE Endpoint

Extension popup shows list of available TEEs:
```
Available Ghost Witness Servers:
○ Official Phala TEE (US) - Recommended
○ Community TEE EU (Europe)
○ Custom endpoint...
```

### 3. Download Verification

When user downloads a file:
```javascript
// Get selected TEE
const tee = registry.tees.find(t => t.endpoint === selectedEndpoint);

// Call TEE for verification
const proof = await fetch(`${tee.endpoint}/api/download`, {
  method: 'POST',
  body: JSON.stringify(downloadData)
});
```

### 4. Proof Verification

When verifying a proof:
```javascript
async function verifyProof(proof) {
  const registry = await loadRegistry();
  
  // Check epoch matches
  if (proof.epoch !== registry.epoch) {
    return { valid: false, reason: 'Epoch mismatch' };
  }
  
  // Check witness is approved
  const witness = registry.witnesses.find(
    w => w.address.toLowerCase() === proof.witness_address.toLowerCase()
  );
  
  if (!witness || witness.status !== 'active') {
    return { valid: false, reason: 'Witness not approved' };
  }
  
  // Verify signature...
  return { valid: true };
}
```

## Adding a New Witness

### Prerequisites
1. Deploy Ghost Witness v1.3 server on Phala Network
2. Obtain TEE wallet address from `/identity` endpoint
3. Verify TEE attestation works

### Submission Process

1. **Fork this repository**

2. **Add your witness to the registry:**
```json
{
  "address": "0xYourWitnessAddress",
  "name": "Your Witness Name",
  "operator": "Your Name/Organization",
  "status": "active"
}
```

3. **Add your TEE endpoint:**
```json
{
  "endpoint": "https://your-tee.example.com",
  "witness_address": "0xYourWitnessAddress",
  "name": "Your TEE Name",
  "region": "EU",
  "status": "active",
  "limits": {
    "requests_per_hour": 50,
    "max_file_size_mb": 50
  },
  "features": {
    "ecies_encryption": true,
    "session_cookies": true
  }
}
```

4. **Provide proof of attestation:**

Include in PR description:
- Output from `GET /identity` endpoint
- TEE quote verification
- Witness address matching TEE wallet

5. **Submit Pull Request**

PR will be reviewed by maintainers for:
- Valid TEE attestation
- Endpoint accessibility
- Witness address verification
- Reasonable rate limits

## Epoch Management

### What is an Epoch?

An epoch represents a protocol version. All witnesses in an epoch must:
- Use the same Ghost Witness protocol version
- Generate compatible proofs
- Follow the same report_data structure

### Epoch Transition

When protocol upgrades:
1. New epoch announced (e.g., epoch 1)
2. Witnesses upgrade their servers
3. Registry updated with new epoch
4. Old proofs (epoch 0) become invalid
5. Users update extensions

### Multi-Epoch Support (Future)

```json
{
  "epochs": [
    {
      "epoch": 0,
      "protocol_version": "1.3",
      "status": "deprecated",
      "sunset_date": "2026-01-01",
      "witnesses": [...]
    },
    {
      "epoch": 1,
      "protocol_version": "1.4",
      "status": "active",
      "witnesses": [...]
    }
  ]
}
```

## Registry Updates

### Frequency
- **Routine updates**: Weekly (new witnesses, status changes)
- **Emergency updates**: As needed (security issues, compromised witnesses)
- **Epoch updates**: When protocol upgrades

### Notification
Applications should:
- Check registry every 24 hours
- Cache locally for offline use
- Alert user if selected TEE becomes inactive

## Security Considerations

### Trust Model
- **GitHub as source of truth** - Repository controlled by maintainers
- **Community review** - All PRs publicly visible
- **No single point of failure** - Multiple approved witnesses

### What Registry CANNOT Do
❌ Registry cannot forge proofs (cryptography enforced by TEE)
❌ Registry cannot modify past proofs (immutable once signed)
❌ Registry cannot access user data (TEE sealed keys)

### What Registry CAN Do
✅ Add/remove witnesses (social consensus)
✅ Deprecate compromised witnesses
✅ Coordinate protocol upgrades
✅ Provide service discovery

## Migration to On-Chain (Future)

This GitHub registry is interim. Future on-chain registry:

```solidity
contract GhostWitnessRegistry {
    struct Witness {
        address addr;
        string name;
        uint256 stake;
        bool active;
    }
    
    mapping(uint256 => Witness[]) public epochWitnesses;
    
    function registerWitness(uint256 epoch, string memory name) 
        external payable {
        require(msg.value >= MIN_STAKE);
        // ...
    }
}
```

**Timeline:** Q2 2026

## Contact

- **Issues:** https://github.com/rachdiamine/templates/issues
- **Discussions:** https://github.com/rachdiamine/templates/discussions
- **Email:** amine.rachdi@realtimetypeapprovals.com

## License

CC0 1.0 Universal - Public Domain
