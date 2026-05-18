# ADR-006: Infrastructure-Only Resource Location

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: Code Review

---

## Context

Domain objects often need to reference files (images, documents, videos). Hardcoding URLs or storage paths in Domain couples it to specific storage backends (S3, Azure Blob, local disk).

---

## Decision

Domain knows only **logical `ResourceId`**. `ResourcePhysicalLocation` and URL generation are **Infrastructure** concerns, resolved at the edge (Adapter/Controller) or just-in-time.

### Constraints

- Domain **MUST** use logical identifiers only (e.g., `ResourceId`, `ImageId`)
- Domain **MUST NOT** know about storage backend (S3, Azure, local)
- URL generation happens in Infrastructure layer
- Physical location resolution is just-in-time (when needed for display/download)

### Boundary Enforcement Extension (API Layer)

`ResourcePhysicalLocation` **MUST NOT** be exposed outside Infrastructure layer, including but not limited to:
- Application services return types
- DTOs (API layer)
- External contracts

Only logical identifiers (`ResourceId`) or access abstractions
(`ResourceAccessToken`, `ResourceReference`) are allowed to cross boundaries.

---

## Consequences

### Positive
✅ Domain is agnostic of storage backend  
✅ Storage migration (S3 → Azure) doesn't touch domain code  
✅ Easy to change URL generation strategy (CDN, signed URLs)  
✅ Domain remains testable without storage infrastructure

### Negative
⚠️ Requires mapping at Infrastructure boundary  
⚠️ Cannot see full URL in domain objects (less convenient)

---

## Example

### ❌ WRONG
```java
// Domain
public class Product {
    private String imageUrl; // ← VIOLATION: Full S3 URL
    
    public Product(String name, String imageUrl) {
        this.name = name;
        this.imageUrl = "https://s3.amazonaws.com/bucket/" + imageUrl; // ← Hardcoded
    }
    
    public String getImageUrl() {
        return this.imageUrl;
    }
}
```

### ✅ CORRECT
```java
// Domain (Logical ID only)
public class Product {
    private ResourceId imageId; // ✅ Logical identifier
    
    public Product(String name, ResourceId imageId) {
        this.name = name;
        this.imageId = imageId;
    }
    
    public ResourceId getImageId() {
        return this.imageId;
    }
}

// Infrastructure (URL generation)
@Component
public class ResourceUrlResolver {
    private final String cdnBaseUrl;
    
    public String resolveUrl(ResourceId resourceId) {
        // Can switch between S3, Azure, CDN without touching domain
        return cdnBaseUrl + "/images/" + resourceId.getValue();
    }
}

// Controller (Edge - generates URLs for response)
@RestController
public class ProductController {
    private final ResourceUrlResolver urlResolver;
    
    @GetMapping("/products/{id}")
    public ProductDTO getProduct(@PathVariable String id) {
        Product product = productService.findById(id);
        
        return new ProductDTO(
            product.getName(),
            urlResolver.resolveUrl(product.getImageId()) // ✅ URL at edge
        );
    }
}
```

---

## Migration Example

```java
// Before: S3
public String resolveUrl(ResourceId resourceId) {
    return "https://s3.amazonaws.com/bucket/" + resourceId.getValue();
}

// After: Azure Blob (NO domain changes needed)
public String resolveUrl(ResourceId resourceId) {
    return "https://mystorageaccount.blob.core.windows.net/images/" + resourceId.getValue();
}
```

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Store full S3 URLs in Domain Entities" | Couples domain to storage backend |
| "Generate URLs in Domain" | Violates single responsibility |
| "Use relative paths in Domain" | Still couples to file system structure |

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-002](ADR-002-infrastructure-only-shared-services.md): Infrastructure-Only Shared Services
