# Zero-Trust Security Implementation Guide - Part 3: Data Security & Encryption

## 1. Data Security Architecture

```plaintext
├── Encryption Service
│   ├── Key Management
│   ├── Encryption Operations
│   └── Key Rotation
├── Data Access Control
│   ├── Data Classification
│   ├── Access Policies
│   └── Audit Logging
└── Data Protection
    ├── Data Masking
    ├── Tokenization
    └── Secure Storage
```

## 2. Implementation Components

### 2.1 Encryption Service

```javascript
// Encryption Service
class EncryptionService {
  properties:
    keyManager: KeyManager
    cryptoProvider: CryptoProvider
    config: EncryptionConfig

  methods:
    async encrypt(data, context):
      try {
        // Get encryption key
        key = await this.keyManager.getKey(context.keyId)
        
        // Generate IV
        iv = this.cryptoProvider.generateIV()
        
        // Encrypt data
        encrypted = await this.cryptoProvider.encrypt(
          data,
          key,
          iv,
          context.algorithm || this.config.defaultAlgorithm
        )
        
        return {
          data: encrypted,
          iv: iv.toString('base64'),
          keyId: context.keyId,
          algorithm: context.algorithm
        }
      } catch (error) {
        this.logger.error('Encryption failed', error)
        throw new EncryptionError('Failed to encrypt data')
      }

    async decrypt(encryptedData, context):
      try {
        // Get decryption key
        key = await this.keyManager.getKey(encryptedData.keyId)
        
        // Decrypt data
        decrypted = await this.cryptoProvider.decrypt(
          encryptedData.data,
          key,
          Buffer.from(encryptedData.iv, 'base64'),
          encryptedData.algorithm
        )
        
        return decrypted
      } catch (error) {
        this.logger.error('Decryption failed', error)
        throw new DecryptionError('Failed to decrypt data')
      }
}

// Key Manager
class KeyManager {
  properties:
    keyStore: KeyStore
    rotationScheduler: KeyRotationScheduler
    config: KeyManagementConfig

  methods:
    async generateKey(context):
      // Generate new key
      key = await this.cryptoProvider.generateKey(
        context.algorithm,
        context.length
      )
      
      // Store key
      keyId = await this.keyStore.store(key, {
        algorithm: context.algorithm,
        created: Date.now(),
        expiresAt: Date.now() + this.config.keyLifetime,
        purpose: context.purpose
      })
      
      return keyId

    async rotateKey(keyId):
      // Get old key metadata
      oldKey = await this.keyStore.getMetadata(keyId)
      
      // Generate new key
      newKeyId = await this.generateKey({
        algorithm: oldKey.algorithm,
        length: oldKey.length,
        purpose: oldKey.purpose
      })
      
      // Mark old key as rotating
      await this.keyStore.updateStatus(keyId, 'rotating')
      
      return {
        oldKeyId: keyId,
        newKeyId: newKeyId
      }
}
```

### 2.2 Data Access Control

```javascript
// Data Access Controller
class DataAccessController {
  properties:
    policyEngine: PolicyEngine
    classifier: DataClassifier
    auditLogger: AuditLogger

  methods:
    async checkAccess(user, data, action):
      try {
        // Classify data
        classification = await this.classifier.classify(data)
        
        // Build access context
        context = {
          user,
          data: {
            id: data.id,
            type: data.type,
            classification
          },
          action,
          timestamp: Date.now()
        }
        
        // Evaluate access
        decision = await this.policyEngine.evaluate(context)
        
        // Log access attempt
        await this.auditLogger.logAccess(context, decision)
        
        return decision
      } catch (error) {
        this.logger.error('Access check failed', error)
        throw new AccessControlError('Failed to check access')
      }

    async maskSensitiveData(data, user):
      // Get data classification
      classification = await this.classifier.classify(data)
      
      // Get masking rules
      rules = await this.getMaskingRules(classification, user)
      
      // Apply masking
      return this.applyMaskingRules(data, rules)
}

// Data Classifier
class DataClassifier {
  properties:
    patterns: Map<string, RegExp>
    rules: ClassificationRules

  methods:
    async classify(data):
      classifications = new Set()
      
      // Check patterns
      for (const [type, pattern] of this.patterns) {
        if (this.matchesPattern(data, pattern)) {
          classifications.add(type)
        }
      }
      
      // Apply rules
      for (const rule of this.rules) {
        if (await rule.evaluate(data)) {
          classifications.add(rule.classification)
        }
      }
      
      return Array.from(classifications)

    matchesPattern(data, pattern):
      if (typeof data === 'string') {
        return pattern.test(data)
      }
      
      if (typeof data === 'object') {
        return Object.values(data).some(value => 
          this.matchesPattern(value, pattern)
        )
      }
      
      return false
}
```

### 2.3 Data Protection

```javascript
// Data Protection Service
class DataProtectionService {
  properties:
    encryptionService: EncryptionService
    tokenizer: Tokenizer
    storageManager: SecureStorageManager

  methods:
    async protectData(data, context):
      try {
        // Classify data
        classification = await this.classifier.classify(data)
        
        // Determine protection strategy
        strategy = this.getProtectionStrategy(classification)
        
        // Apply protection
        protected = await this.applyProtection(data, strategy)
        
        // Store protection metadata
        await this.storageManager.storeMetadata(protected.id, {
          strategy,
          classification,
          timestamp: Date.now()
        })
        
        return protected
      } catch (error) {
        this.logger.error('Data protection failed', error)
        throw new ProtectionError('Failed to protect data')
      }

    async applyProtection(data, strategy):
      switch (strategy.type) {
        case 'encrypt':
          return this.encryptionService.encrypt(data, strategy.config)
        
        case 'tokenize':
          return this.tokenizer.tokenize(data, strategy.config)
        
        case 'mask':
          return this.maskData(data, strategy.config)
        
        default:
          throw new Error(`Unknown protection strategy: ${strategy.type}`)
      }

    getProtectionStrategy(classification):
      return this.config.protectionStrategies
        .find(strategy => 
          strategy.classifications.includes(classification)
        )
}
```

## Related Topics
- [[Zero-Trust-Security-Part1]] - Core Architecture
- [[Zero-Trust-Security-Part2]] - Network Security & Microsegmentation
- [[Zero-Trust-Security-Part4]] - Monitoring & Response

## Tags
#security #zero-trust #encryption #data-security #access-control
