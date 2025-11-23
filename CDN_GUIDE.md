# CDN (Content Delivery Network) - Complete Implementation Guide

## Table of Contents
1. [What is CDN?](#what-is-cdn)
2. [How CDN Works](#how-cdn-works)
3. [When to Use CDN](#when-to-use-cdn)
4. [CDN Use Cases and Examples](#cdn-use-cases-and-examples)
5. [CDN Implementation](#cdn-implementation)
6. [Real-World Examples](#real-world-examples)
7. [Best Practices](#best-practices)
8. [CDN Providers Comparison](#cdn-providers-comparison)
9. [Strategies and Patterns](#strategies-and-patterns)
10. [Interview Questions](#interview-questions)

---

## What is CDN?

**CDN (Content Delivery Network)** is a distributed network of servers that deliver web content to users based on their geographic location, providing faster and more reliable content delivery.

### Key Concepts:

- **Edge Servers**: Servers located close to end users
- **Origin Server**: Original server where content is stored
- **Caching**: Storing content on edge servers for faster access
- **Geographic Distribution**: Servers in multiple locations worldwide

### How CDN Improves Performance:

**Without CDN:**
```
User (India) → Origin Server (USA)
Distance: 8,000 miles
Latency: 200-300ms
Bandwidth: Limited by origin server
```

**With CDN:**
```
User (India) → Edge Server (Mumbai)
Distance: 50 miles
Latency: 10-20ms
Bandwidth: Optimized for content delivery
```

### Benefits of CDN:

1. **Faster Load Times**: Reduced latency (10-20ms vs 200-300ms)
2. **Reduced Server Load**: Offloads traffic from origin server
3. **Better Availability**: Multiple edge servers (redundancy)
4. **Cost Savings**: Reduced bandwidth costs
5. **Global Reach**: Consistent performance worldwide
6. **DDoS Protection**: Built-in security features
7. **Scalability**: Handles traffic spikes automatically

---

## How CDN Works

### CDN Architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    CDN ARCHITECTURE                      │
└─────────────────────────────────────────────────────────┘

User Request
    │
    ├─→ Check Edge Server (Mumbai)
    │   │
    │   ├─→ Cache Hit? ──Yes──→ Return Cached Content (10ms)
    │   │
    │   └─→ Cache Miss? ──No──→ Request from Origin
    │                           │
    │                           ├─→ Origin Server (USA)
    │                           │   │
    │                           │   └─→ Fetch Content (200ms)
    │                           │
    │                           └─→ Cache on Edge Server
    │                           │
    │                           └─→ Return to User
    │
    └─→ User Receives Content
```

### CDN Flow:

1. **User Request**: User requests content (e.g., image, CSS file)
2. **DNS Resolution**: DNS routes to nearest edge server
3. **Edge Server Check**: Edge server checks cache
4. **Cache Hit**: If cached, return immediately
5. **Cache Miss**: If not cached, fetch from origin
6. **Cache Update**: Store in edge server cache
7. **Response**: Return content to user

### CDN Caching Strategies:

**1. Cache-Control Headers:**
```
Cache-Control: public, max-age=3600
```
- `public`: Can be cached by CDN
- `max-age=3600`: Cache for 1 hour

**2. Cache Invalidation:**
- **TTL (Time To Live)**: Automatic expiration
- **Manual Purge**: Force cache refresh
- **Versioning**: URL-based versioning (`style.css?v=2.0`)

**3. Cache Levels:**
- **Browser Cache**: User's browser
- **CDN Cache**: Edge servers
- **Origin Cache**: Origin server cache

---

## When to Use CDN

### ✅ Use CDN When:

1. **Static Assets**
   - Images (product photos, banners)
   - CSS files
   - JavaScript files
   - Fonts
   - Videos
   - Documents (PDFs, downloads)

2. **Global Audience**
   - Users in multiple countries
   - Need consistent performance worldwide
   - High latency to origin server

3. **High Traffic**
   - Millions of requests per day
   - Traffic spikes (sales, events)
   - Bandwidth-intensive content

4. **Performance Critical**
   - Page load time matters
   - User experience is priority
   - SEO optimization

5. **Cost Optimization**
   - Reduce origin server bandwidth
   - Lower hosting costs
   - Pay for what you use

### ❌ Don't Use CDN When:

1. **Dynamic Content**
   - User-specific data
   - Real-time updates
   - Personalized content

2. **Small Audience**
   - Local users only
   - Low traffic
   - Cost not justified

3. **Sensitive Data**
   - Private user data
   - Authentication tokens
   - Financial information

4. **Frequently Changing Content**
   - Content changes every minute
   - Cache invalidation overhead
   - Better served from origin

---

## CDN Use Cases and Examples

### 1. Static Web Files (HTML, CSS, JavaScript)

**Use Case:** Serve frontend assets (React, Angular, Vue apps)

**Example: React Application**

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <!-- CSS from CDN -->
    <link rel="stylesheet" href="https://cdn.example.com/css/bootstrap.min.css">
    <link rel="stylesheet" href="https://cdn.example.com/css/app.css">
</head>
<body>
    <div id="root"></div>
    
    <!-- JavaScript from CDN -->
    <script src="https://cdn.example.com/js/react.production.min.js"></script>
    <script src="https://cdn.example.com/js/react-dom.production.min.js"></script>
    <script src="https://cdn.example.com/js/app.bundle.js"></script>
</body>
</html>
```

**Implementation:**

```java
// Spring Boot Configuration
@Configuration
public class CDNConfig {
    
    @Value("${cdn.base-url:https://cdn.example.com}")
    private String cdnBaseUrl;
    
    @Bean
    public String cdnBaseUrl() {
        return cdnBaseUrl;
    }
}

// Service to generate CDN URLs
@Service
public class CDNService {
    
    @Autowired
    private String cdnBaseUrl;
    
    public String getStaticFileUrl(String path) {
        return cdnBaseUrl + "/static/" + path;
    }
    
    public String getCSSUrl(String filename) {
        return cdnBaseUrl + "/css/" + filename + "?v=" + getVersion();
    }
    
    public String getJSUrl(String filename) {
        return cdnBaseUrl + "/js/" + filename + "?v=" + getVersion();
    }
    
    private String getVersion() {
        // Use application version or timestamp
        return "2.0.1";
    }
}
```

**Frontend Integration:**

```javascript
// React Component
import React from 'react';

const CDN_BASE_URL = 'https://cdn.example.com';

function App() {
    return (
        <div>
            <img src={`${CDN_BASE_URL}/images/logo.png`} alt="Logo" />
            <link rel="stylesheet" href={`${CDN_BASE_URL}/css/app.css?v=2.0.1`} />
        </div>
    );
}
```

---

### 2. Product Images

**Use Case:** Serve product images for e-commerce

**Current Implementation (Without CDN):**

```java
// ProductService.java - Current implementation
private String generateProductImageUrl(String productName, String brand, Long categoryId, Long skuId) {
    // Currently using external service (Picsum Photos)
    return String.format("https://picsum.photos/seed/%s%d/600/600", imageCategory, seed);
}
```

**Improved Implementation (With CDN):**

```java
@Service
public class ProductImageService {
    
    @Value("${cdn.images.base-url:https://cdn.example.com}")
    private String cdnBaseUrl;
    
    @Value("${cdn.images.version:1.0}")
    private String imageVersion;
    
    /**
     * Generate CDN URL for product image
     */
    public String getProductImageUrl(Long skuId, String imageName, ImageSize size) {
        // CDN URL structure: https://cdn.example.com/images/products/{skuId}/{size}/{imageName}
        return String.format("%s/images/products/%d/%s/%s?v=%s",
            cdnBaseUrl, skuId, size.getValue(), imageName, imageVersion);
    }
    
    /**
     * Generate multiple image URLs (thumbnail, medium, large)
     */
    public ProductImageUrls getProductImageUrls(Long skuId, String imageName) {
        ProductImageUrls urls = new ProductImageUrls();
        urls.setThumbnail(getProductImageUrl(skuId, imageName, ImageSize.THUMBNAIL));
        urls.setMedium(getProductImageUrl(skuId, imageName, ImageSize.MEDIUM));
        urls.setLarge(getProductImageUrl(skuId, imageName, ImageSize.LARGE));
        return urls;
    }
    
    /**
     * Upload product image to CDN
     */
    public String uploadProductImage(Long skuId, MultipartFile imageFile) {
        try {
            // 1. Validate image
            validateImage(imageFile);
            
            // 2. Process image (resize, optimize)
            byte[] processedImage = processImage(imageFile);
            
            // 3. Upload to CDN storage (S3, CloudFront, etc.)
            String imageName = generateImageName(skuId, imageFile.getOriginalFilename());
            String cdnUrl = uploadToCDN(processedImage, imageName);
            
            // 4. Store metadata in database
            saveImageMetadata(skuId, imageName, cdnUrl);
            
            return cdnUrl;
        } catch (Exception e) {
            throw new ImageUploadException("Failed to upload image", e);
        }
    }
    
    private void validateImage(MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("Image file is empty");
        }
        
        String contentType = file.getContentType();
        if (contentType == null || !contentType.startsWith("image/")) {
            throw new IllegalArgumentException("File is not an image");
        }
        
        // Check file size (max 5MB)
        if (file.getSize() > 5 * 1024 * 1024) {
            throw new IllegalArgumentException("Image size exceeds 5MB");
        }
    }
    
    private byte[] processImage(MultipartFile file) throws IOException {
        // Resize and optimize image
        // Use libraries like Thumbnailator or ImageIO
        return file.getBytes();
    }
    
    private String uploadToCDN(byte[] imageData, String imageName) {
        // Upload to S3, CloudFront, or other CDN storage
        // Implementation depends on CDN provider
        return cdnBaseUrl + "/images/products/" + imageName;
    }
    
    public enum ImageSize {
        THUMBNAIL("thumbnail", 150, 150),
        SMALL("small", 300, 300),
        MEDIUM("medium", 600, 600),
        LARGE("large", 1200, 1200);
        
        private final String value;
        private final int width;
        private final int height;
        
        ImageSize(String value, int width, int height) {
            this.value = value;
            this.width = width;
            this.height = height;
        }
        
        public String getValue() { return value; }
        public int getWidth() { return width; }
        public int getHeight() { return height; }
    }
    
    public static class ProductImageUrls {
        private String thumbnail;
        private String medium;
        private String large;
        
        // Getters and setters
    }
}
```

**DTO Update:**

```java
// ProductDTO.java
public class ProductDTO {
    private Long skuId;
    private String productName;
    
    // Single image URL (backward compatible)
    private String imageUrl;
    
    // Multiple image URLs (CDN optimized)
    private ProductImageUrls imageUrls;
    
    // Getters and setters
}
```

**Controller:**

```java
@RestController
@RequestMapping("/api/products")
public class ProductImageController {
    
    @Autowired
    private ProductImageService imageService;
    
    @PostMapping("/{skuId}/images")
    public ResponseEntity<String> uploadProductImage(
            @PathVariable Long skuId,
            @RequestParam("image") MultipartFile imageFile) {
        String imageUrl = imageService.uploadProductImage(skuId, imageFile);
        return ResponseEntity.ok(imageUrl);
    }
    
    @GetMapping("/{skuId}/images")
    public ResponseEntity<ProductImageUrls> getProductImages(@PathVariable Long skuId) {
        ProductImageUrls urls = imageService.getProductImageUrls(skuId, "main.jpg");
        return ResponseEntity.ok(urls);
    }
}
```

**Frontend Usage:**

```javascript
// React Component
function ProductCard({ product }) {
    const imageUrl = product.imageUrls?.medium || product.imageUrl;
    
    return (
        <div className="product-card">
            <img 
                src={imageUrl} 
                alt={product.productName}
                loading="lazy"  // Lazy loading
                srcSet={`${product.imageUrls?.thumbnail} 150w,
                         ${product.imageUrls?.medium} 600w,
                         ${product.imageUrls?.large} 1200w`}
                sizes="(max-width: 600px) 300px, 600px"
            />
        </div>
    );
}
```

---

### 3. File Downloads

**Use Case:** Serve downloadable files (PDFs, documents, software)

**Implementation:**

```java
@Service
public class FileDownloadService {
    
    @Value("${cdn.downloads.base-url:https://cdn.example.com}")
    private String cdnBaseUrl;
    
    @Autowired
    private FileStorageService storageService;
    
    /**
     * Generate CDN URL for file download
     */
    public String getDownloadUrl(String fileId, String fileName) {
        // CDN URL with expiration token for security
        String token = generateSecureToken(fileId);
        return String.format("%s/downloads/%s?file=%s&token=%s",
            cdnBaseUrl, fileId, fileName, token);
    }
    
    /**
     * Generate signed URL (expires in 1 hour)
     */
    public String getSignedDownloadUrl(String fileId, String fileName, Duration expiration) {
        // Generate signed URL that expires after specified duration
        long expiresAt = System.currentTimeMillis() + expiration.toMillis();
        String signature = generateSignature(fileId, expiresAt);
        
        return String.format("%s/downloads/%s?file=%s&expires=%d&signature=%s",
            cdnBaseUrl, fileId, fileName, expiresAt, signature);
    }
    
    /**
     * Upload file to CDN
     */
    public String uploadFile(MultipartFile file, String category) {
        try {
            // 1. Validate file
            validateFile(file);
            
            // 2. Generate unique file ID
            String fileId = UUID.randomUUID().toString();
            String fileName = sanitizeFileName(file.getOriginalFilename());
            
            // 3. Upload to CDN storage
            String cdnUrl = storageService.uploadFile(file.getBytes(), fileId, fileName);
            
            // 4. Store metadata
            FileMetadata metadata = new FileMetadata();
            metadata.setFileId(fileId);
            metadata.setFileName(fileName);
            metadata.setCdnUrl(cdnUrl);
            metadata.setCategory(category);
            metadata.setFileSize(file.getSize());
            metadata.setContentType(file.getContentType());
            metadata.setUploadedAt(Instant.now());
            saveFileMetadata(metadata);
            
            return cdnUrl;
        } catch (Exception e) {
            throw new FileUploadException("Failed to upload file", e);
        }
    }
    
    /**
     * Download file (with CDN fallback)
     */
    public ResponseEntity<Resource> downloadFile(String fileId) {
        // 1. Get file metadata
        FileMetadata metadata = getFileMetadata(fileId);
        if (metadata == null) {
            throw new FileNotFoundException("File not found: " + fileId);
        }
        
        // 2. Try CDN first
        try {
            return ResponseEntity
                .status(HttpStatus.FOUND)
                .header(HttpHeaders.LOCATION, metadata.getCdnUrl())
                .build();
        } catch (Exception e) {
            // 3. Fallback to origin server
            Resource file = storageService.getFile(fileId);
            return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, 
                    "attachment; filename=\"" + metadata.getFileName() + "\"")
                .contentType(MediaType.parseMediaType(metadata.getContentType()))
                .body(file);
        }
    }
    
    private String generateSecureToken(String fileId) {
        // Generate secure token for file access
        String secret = "your-secret-key";
        String data = fileId + System.currentTimeMillis();
        return HmacUtils.hmacSha256Hex(secret, data);
    }
    
    private String generateSignature(String fileId, long expiresAt) {
        // Generate signature for signed URL
        String message = fileId + expiresAt;
        return HmacUtils.hmacSha256Hex("your-secret-key", message);
    }
}
```

**Controller:**

```java
@RestController
@RequestMapping("/api/files")
public class FileDownloadController {
    
    @Autowired
    private FileDownloadService fileService;
    
    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("category") String category) {
        String cdnUrl = fileService.uploadFile(file, category);
        
        Map<String, String> response = new HashMap<>();
        response.put("url", cdnUrl);
        response.put("message", "File uploaded successfully");
        
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/download/{fileId}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String fileId) {
        return fileService.downloadFile(fileId);
    }
    
    @GetMapping("/download/{fileId}/signed")
    public ResponseEntity<Map<String, String>> getSignedDownloadUrl(
            @PathVariable String fileId) {
        String signedUrl = fileService.getSignedDownloadUrl(
            fileId, "document.pdf", Duration.ofHours(1));
        
        Map<String, String> response = new HashMap<>();
        response.put("url", signedUrl);
        response.put("expiresIn", "3600"); // seconds
        
        return ResponseEntity.ok(response);
    }
}
```

---

### 4. Video Content

**Use Case:** Serve video content (product videos, tutorials)

**Implementation:**

```java
@Service
public class VideoCDNService {
    
    @Value("${cdn.videos.base-url:https://cdn.example.com}")
    private String cdnBaseUrl;
    
    /**
     * Generate video CDN URL with different qualities
     */
    public VideoUrls getVideoUrls(String videoId) {
        VideoUrls urls = new VideoUrls();
        urls.setHls(getHlsUrl(videoId));
        urls.setDash(getDashUrl(videoId));
        urls.setThumbnail(getThumbnailUrl(videoId));
        urls.setPoster(getPosterUrl(videoId));
        return urls;
    }
    
    private String getHlsUrl(String videoId) {
        // HLS (HTTP Live Streaming) URL
        return String.format("%s/videos/%s/playlist.m3u8", cdnBaseUrl, videoId);
    }
    
    private String getDashUrl(String videoId) {
        // DASH (Dynamic Adaptive Streaming) URL
        return String.format("%s/videos/%s/manifest.mpd", cdnBaseUrl, videoId);
    }
    
    private String getThumbnailUrl(String videoId) {
        return String.format("%s/videos/%s/thumbnail.jpg", cdnBaseUrl, videoId);
    }
    
    private String getPosterUrl(String videoId) {
        return String.format("%s/videos/%s/poster.jpg", cdnBaseUrl, videoId);
    }
    
    public static class VideoUrls {
        private String hls;
        private String dash;
        private String thumbnail;
        private String poster;
        
        // Getters and setters
    }
}
```

**Frontend Usage:**

```html
<!-- HTML5 Video with CDN -->
<video 
    controls 
    poster="https://cdn.example.com/videos/123/poster.jpg"
    preload="metadata">
    <source src="https://cdn.example.com/videos/123/playlist.m3u8" type="application/x-mpegURL">
    <source src="https://cdn.example.com/videos/123/manifest.mpd" type="application/dash+xml">
    Your browser does not support the video tag.
</video>
```

---

### 5. Fonts and Web Fonts

**Use Case:** Serve custom fonts (Google Fonts alternative)

**Implementation:**

```java
@Service
public class FontCDNService {
    
    @Value("${cdn.fonts.base-url:https://cdn.example.com}")
    private String cdnBaseUrl;
    
    /**
     * Generate font CDN URL
     */
    public String getFontUrl(String fontFamily, String weight, String style) {
        String fontName = sanitizeFontName(fontFamily);
        return String.format("%s/fonts/%s-%s-%s.woff2",
            cdnBaseUrl, fontName, weight, style);
    }
    
    /**
     * Generate @font-face CSS
     */
    public String generateFontFaceCSS(String fontFamily, String weight, String style) {
        String fontUrl = getFontUrl(fontFamily, weight, style);
        return String.format(
            "@font-face {\n" +
            "  font-family: '%s';\n" +
            "  src: url('%s') format('woff2');\n" +
            "  font-weight: %s;\n" +
            "  font-style: %s;\n" +
            "}",
            fontFamily, fontUrl, weight, style
        );
    }
}
```

**CSS Usage:**

```css
/* fonts.css served from CDN */
@font-face {
    font-family: 'CustomFont';
    src: url('https://cdn.example.com/fonts/customfont-400-normal.woff2') format('woff2');
    font-weight: 400;
    font-style: normal;
}

@font-face {
    font-family: 'CustomFont';
    src: url('https://cdn.example.com/fonts/customfont-700-normal.woff2') format('woff2');
    font-weight: 700;
    font-style: normal;
}
```

---

### 6. API Response Caching

**Use Case:** Cache API responses on CDN edge servers

**Implementation:**

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping
    @Cacheable(value = "products", key = "#page + '-' + #size")
    public ResponseEntity<ApiResponse<List<ProductDTO>>> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        ApiResponse<List<ProductDTO>> response = productService.getAllProducts(page, size);
        
        // Set CDN cache headers
        HttpHeaders headers = new HttpHeaders();
        headers.setCacheControl("public, max-age=3600"); // Cache for 1 hour
        headers.set("CDN-Cache-Control", "public, max-age=3600");
        
        return ResponseEntity.ok()
            .headers(headers)
            .body(response);
    }
    
    @GetMapping("/{skuId}")
    public ResponseEntity<ApiResponse<ProductDTO>> getProduct(@PathVariable Long skuId) {
        ApiResponse<ProductDTO> response = productService.getProductBySkuId(skuId);
        
        HttpHeaders headers = new HttpHeaders();
        // Cache product details for 5 minutes
        headers.setCacheControl("public, max-age=300");
        headers.setETag("\"" + skuId + "-" + response.getData().getUpdatedAt() + "\"");
        
        return ResponseEntity.ok()
            .headers(headers)
            .body(response);
    }
}
```

**CDN Configuration (CloudFront example):**

```yaml
# application.yaml
cdn:
  cloudfront:
    distribution-id: E1234567890ABC
    domain: d1234567890.cloudfront.net
    cache-behaviors:
      - path-pattern: "/api/products/*"
        ttl: 3600  # 1 hour
        allowed-methods: ["GET", "HEAD", "OPTIONS"]
      - path-pattern: "/api/products/{skuId}"
        ttl: 300   # 5 minutes
        allowed-methods: ["GET", "HEAD", "OPTIONS"]
```

---

## CDN Implementation

### 1. AWS CloudFront Setup

**Step 1: Create S3 Bucket for Origin**

```bash
# Create S3 bucket
aws s3 mb s3://ecommerce-cdn-assets

# Upload files
aws s3 cp ./images s3://ecommerce-cdn-assets/images --recursive
aws s3 cp ./css s3://ecommerce-cdn-assets/css --recursive
aws s3 cp ./js s3://ecommerce-cdn-assets/js --recursive
```

**Step 2: Create CloudFront Distribution**

```java
@Service
public class CloudFrontService {
    
    @Autowired
    private AmazonCloudFrontClient cloudFrontClient;
    
    public String createDistribution(String originDomain) {
        DistributionConfig config = new DistributionConfig()
            .withCallerReference(UUID.randomUUID().toString())
            .withComment("E-commerce CDN Distribution")
            .withEnabled(true)
            .withDefaultCacheBehavior(createDefaultCacheBehavior(originDomain))
            .withOrigins(createOrigins(originDomain));
        
        CreateDistributionRequest request = new CreateDistributionRequest()
            .withDistributionConfig(config);
        
        CreateDistributionResult result = cloudFrontClient.createDistribution(request);
        return result.getDistribution().getDomainName();
    }
    
    private DefaultCacheBehavior createDefaultCacheBehavior(String originDomain) {
        return new DefaultCacheBehavior()
            .withTargetOriginId(originDomain)
            .withViewerProtocolPolicy(ViewerProtocolPolicy.REDIRECT_TO_HTTPS)
            .withAllowedMethods(new AllowedMethods()
                .withItems("GET", "HEAD", "OPTIONS")
                .withCachedMethods(new CachedMethods()
                    .withItems("GET", "HEAD")))
            .withForwardedValues(new ForwardedValues().withQueryString(false))
            .withMinTTL(0L)
            .withDefaultTTL(3600L)
            .withMaxTTL(86400L);
    }
}
```

**Step 3: Spring Boot Configuration**

```yaml
# application.yaml
cdn:
  cloudfront:
    distribution-id: ${CLOUDFRONT_DISTRIBUTION_ID}
    domain: ${CLOUDFRONT_DOMAIN}
    base-url: https://${CLOUDFRONT_DOMAIN}
  s3:
    bucket-name: ${S3_BUCKET_NAME}
    region: ${AWS_REGION:us-east-1}
```

```java
@Configuration
public class AWSConfig {
    
    @Value("${cdn.s3.region}")
    private String region;
    
    @Bean
    public AmazonS3Client s3Client() {
        return (AmazonS3Client) AmazonS3ClientBuilder.standard()
            .withRegion(region)
            .build();
    }
    
    @Bean
    public AmazonCloudFrontClient cloudFrontClient() {
        return (AmazonCloudFrontClient) AmazonCloudFrontClientBuilder.standard()
            .withRegion(region)
            .build();
    }
}
```

---

### 2. Cloudflare CDN Setup

**Step 1: Add Domain to Cloudflare**

```java
@Service
public class CloudflareService {
    
    @Value("${cdn.cloudflare.zone-id}")
    private String zoneId;
    
    @Value("${cdn.cloudflare.api-key}")
    private String apiKey;
    
    @Value("${cdn.cloudflare.email}")
    private String email;
    
    /**
     * Purge cache for specific URL
     */
    public void purgeCache(String url) {
        // Cloudflare API call to purge cache
        // Implementation using Cloudflare API
    }
    
    /**
     * Purge all cache
     */
    public void purgeAllCache() {
        // Purge entire cache
    }
}
```

**Step 2: Configuration**

```yaml
# application.yaml
cdn:
  cloudflare:
    zone-id: ${CLOUDFLARE_ZONE_ID}
    api-key: ${CLOUDFLARE_API_KEY}
    email: ${CLOUDFLARE_EMAIL}
    base-url: https://cdn.yourdomain.com
```

---

## Real-World Examples

### Example 1: E-Commerce Product Images

**Scenario:** Serve product images with different sizes and formats

```java
@Service
public class ProductImageCDNService {
    
    @Value("${cdn.images.base-url}")
    private String cdnBaseUrl;
    
    /**
     * Get optimized image URL with WebP support
     */
    public String getOptimizedImageUrl(Long skuId, ImageSize size) {
        // Check if browser supports WebP
        // Return WebP for modern browsers, JPEG for older browsers
        String format = supportsWebP() ? "webp" : "jpg";
        return String.format("%s/images/products/%d/%s.%s",
            cdnBaseUrl, skuId, size.getValue(), format);
    }
    
    /**
     * Generate responsive image URLs
     */
    public ResponsiveImageUrls getResponsiveImageUrls(Long skuId) {
        ResponsiveImageUrls urls = new ResponsiveImageUrls();
        urls.setSrc(getOptimizedImageUrl(skuId, ImageSize.LARGE));
        urls.setSrcSet(String.format(
            "%s 600w, %s 1200w, %s 2400w",
            getOptimizedImageUrl(skuId, ImageSize.MEDIUM),
            getOptimizedImageUrl(skuId, ImageSize.LARGE),
            getOptimizedImageUrl(skuId, ImageSize.XLARGE)
        ));
        urls.setSizes("(max-width: 600px) 300px, (max-width: 1200px) 600px, 1200px");
        return urls;
    }
}
```

**Frontend:**

```jsx
function ProductImage({ skuId }) {
    const imageUrls = useProductImageUrls(skuId);
    
    return (
        <picture>
            <source 
                srcSet={imageUrls.webp.srcSet} 
                type="image/webp"
                sizes={imageUrls.webp.sizes}
            />
            <source 
                srcSet={imageUrls.jpg.srcSet} 
                type="image/jpeg"
                sizes={imageUrls.jpg.sizes}
            />
            <img 
                src={imageUrls.jpg.src}
                alt="Product"
                loading="lazy"
            />
        </picture>
    );
}
```

---

### Example 2: Static Asset Versioning

**Scenario:** Cache busting for static assets

```java
@Service
public class AssetVersionService {
    
    @Value("${app.version}")
    private String appVersion;
    
    @Value("${cdn.assets.base-url}")
    private String cdnBaseUrl;
    
    /**
     * Generate versioned asset URL
     */
    public String getVersionedAssetUrl(String assetPath) {
        // Option 1: Query parameter versioning
        return String.format("%s/%s?v=%s", cdnBaseUrl, assetPath, appVersion);
        
        // Option 2: Path-based versioning
        // return String.format("%s/v%s/%s", cdnBaseUrl, appVersion, assetPath);
        
        // Option 3: Filename-based versioning
        // return String.format("%s/%s.%s", cdnBaseUrl, assetPath, appVersion);
    }
    
    /**
     * Generate hash-based version (content-based)
     */
    public String getHashedAssetUrl(String assetPath) {
        try {
            byte[] content = loadAssetContent(assetPath);
            String hash = DigestUtils.md5Hex(content).substring(0, 8);
            return String.format("%s/%s?h=%s", cdnBaseUrl, assetPath, hash);
        } catch (Exception e) {
            return getVersionedAssetUrl(assetPath);
        }
    }
}
```

---

## Best Practices

### 1. Cache Headers

**✅ DO:**
```java
// Set appropriate cache headers
headers.setCacheControl("public, max-age=3600, immutable");
headers.setETag("\"" + contentHash + "\"");
headers.setLastModified(lastModified);
```

**❌ DON'T:**
```java
// Don't cache dynamic content
headers.setCacheControl("no-cache, no-store, must-revalidate");
```

### 2. Image Optimization

**✅ DO:**
- Use WebP format for modern browsers
- Provide multiple sizes (thumbnail, medium, large)
- Lazy load images
- Use responsive images (srcset)

**❌ DON'T:**
- Serve full-size images to mobile
- Use uncompressed images
- Load all images at once

### 3. CDN URL Structure

**✅ GOOD:**
```
https://cdn.example.com/images/products/123/medium.jpg
https://cdn.example.com/css/app.v2.0.1.css
https://cdn.example.com/js/bundle.v2.0.1.js
```

**❌ BAD:**
```
https://cdn.example.com/files/abc123def456.jpg
https://cdn.example.com/assets/style.css
```

### 4. Security

**✅ DO:**
- Use signed URLs for private content
- Set appropriate CORS headers
- Validate file types
- Rate limit uploads

**❌ DON'T:**
- Expose sensitive files
- Allow arbitrary file uploads
- Skip authentication for uploads

---

## CDN Providers Comparison

| Provider | Pros | Cons | Best For |
|----------|------|------|----------|
| **AWS CloudFront** | Integrated with AWS, global network, DDoS protection | Can be expensive, AWS lock-in | AWS ecosystem |
| **Cloudflare** | Free tier, DDoS protection, easy setup | Limited customization on free tier | Small to medium sites |
| **Fastly** | Real-time purging, edge computing | Expensive, complex setup | High-performance needs |
| **Akamai** | Largest network, enterprise features | Very expensive | Enterprise |
| **Azure CDN** | Integrated with Azure, good pricing | Smaller network than AWS | Azure ecosystem |
| **Google Cloud CDN** | Integrated with GCP, good performance | Smaller network | GCP ecosystem |

---

## Strategies and Patterns

### 1. Multi-CDN Strategy

**Use multiple CDNs for redundancy:**

```java
@Service
public class MultiCDNService {
    
    @Value("${cdn.primary.url}")
    private String primaryCDN;
    
    @Value("${cdn.fallback.url}")
    private String fallbackCDN;
    
    public String getAssetUrl(String assetPath) {
        // Try primary CDN first
        if (isCDNAvailable(primaryCDN)) {
            return primaryCDN + "/" + assetPath;
        }
        // Fallback to secondary CDN
        return fallbackCDN + "/" + assetPath;
    }
}
```

### 2. CDN with Origin Fallback

```java
@Service
public class CDNWithFallbackService {
    
    @Value("${cdn.base-url}")
    private String cdnUrl;
    
    @Value("${app.origin-url}")
    private String originUrl;
    
    public String getAssetUrl(String assetPath) {
        // Always try CDN first
        String cdnAssetUrl = cdnUrl + "/" + assetPath;
        
        // If CDN fails, fallback to origin
        if (!isCDNAvailable(cdnUrl)) {
            return originUrl + "/" + assetPath;
        }
        
        return cdnAssetUrl;
    }
}
```

### 3. Progressive Enhancement

```html
<!-- Load from CDN with fallback -->
<script src="https://cdn.example.com/js/app.js"></script>
<script>
    // Fallback if CDN fails
    if (typeof App === 'undefined') {
        document.write('<script src="/js/app.js"><\/script>');
    }
</script>
```

---

## Interview Questions

### Q1: What is CDN and why use it?

**Answer:**
"A CDN (Content Delivery Network) is a distributed network of servers that deliver web content to users based on their geographic location. I use CDN to:
1. **Reduce latency**: Serve content from edge servers close to users (10-20ms vs 200-300ms)
2. **Reduce server load**: Offload traffic from origin server
3. **Improve availability**: Multiple edge servers provide redundancy
4. **Cost savings**: Reduce bandwidth costs on origin server
5. **Global reach**: Consistent performance worldwide

In our e-commerce system, I use CDN for product images, static assets (CSS/JS), and file downloads. This reduces page load time by 60% and improves user experience."

### Q2: How does CDN caching work?

**Answer:**
"CDN caching works in multiple layers:
1. **Edge Server Cache**: Content cached on edge servers closest to users
2. **Cache Headers**: Origin server sets Cache-Control headers (max-age, public, etc.)
3. **Cache Hit**: If content is cached and not expired, return immediately
4. **Cache Miss**: If not cached, fetch from origin, cache it, then return
5. **Cache Invalidation**: Use TTL expiration, manual purge, or versioning

I configure appropriate TTLs: product images (1 hour), static assets (1 day with versioning), API responses (5 minutes). For cache invalidation, I use URL versioning (`app.js?v=2.0.1`) for static assets and manual purge for product images when updated."

### Q3: How do you handle CDN failures?

**Answer:**
"I handle CDN failures with multiple strategies:
1. **Health Checks**: Monitor CDN availability
2. **Fallback to Origin**: If CDN fails, serve from origin server
3. **Multiple CDNs**: Use primary and fallback CDN
4. **Graceful Degradation**: Serve lower quality images or skip non-critical assets
5. **Error Handling**: Log failures and alert monitoring system

In our implementation, I check CDN health and automatically fallback to origin server if CDN is unavailable. I also use multiple CDN providers for critical assets to ensure high availability."

### Q4: How do you optimize images for CDN?

**Answer:**
"I optimize images for CDN by:
1. **Format Selection**: Use WebP for modern browsers, JPEG/PNG for fallback
2. **Multiple Sizes**: Generate thumbnail, medium, large versions
3. **Compression**: Optimize file size without quality loss
4. **Lazy Loading**: Load images only when needed
5. **Responsive Images**: Use srcset for different screen sizes
6. **CDN Transformation**: Use CDN image transformation APIs

I generate three sizes for each product image (150x150, 600x600, 1200x1200) and serve appropriate size based on device. I also use WebP format which reduces file size by 30% compared to JPEG."

### Q5: How do you secure CDN content?

**Answer:**
"I secure CDN content by:
1. **Signed URLs**: Generate time-limited signed URLs for private content
2. **Access Control**: Use CDN access control features
3. **HTTPS**: Always use HTTPS for CDN content
4. **CORS Headers**: Set appropriate CORS headers
5. **Token Validation**: Validate tokens before serving content
6. **Rate Limiting**: Limit requests per IP

For file downloads, I generate signed URLs that expire after 1 hour. For product images, I use public CDN with appropriate cache headers. For sensitive documents, I use signed URLs with authentication."

### Q6: What is the difference between CDN and caching?

**Answer:**
"CDN and caching are related but different:
- **CDN**: Distributed network of servers in multiple locations, serves content from nearest server
- **Caching**: Storing content temporarily for faster access

CDN uses caching as a mechanism - it caches content on edge servers. But CDN also provides:
- Geographic distribution
- Load balancing
- DDoS protection
- Analytics

I use both: CDN for static assets and images (geographic distribution), and application-level caching (Redis) for dynamic data. CDN reduces latency, while Redis reduces database load."

### Q7: How do you implement cache invalidation?

**Answer:**
"I implement cache invalidation using multiple strategies:
1. **TTL-based**: Set expiration time (max-age in Cache-Control)
2. **Versioning**: URL-based versioning (`app.js?v=2.0.1`)
3. **Manual Purge**: CDN API to purge specific URLs
4. **Event-based**: Purge cache when content updates
5. **Hash-based**: Content hash in URL (changes when content changes)

For static assets, I use versioning. For product images, I purge cache when product is updated. For API responses, I use short TTL (5 minutes) with ETag for validation."

### Q8: How do you measure CDN performance?

**Answer:**
"I measure CDN performance using:
1. **Response Time**: Latency from edge server
2. **Cache Hit Ratio**: Percentage of requests served from cache
3. **Bandwidth Savings**: Traffic offloaded from origin
4. **Error Rate**: Failed requests percentage
5. **Geographic Performance**: Performance by region
6. **Real User Monitoring**: Actual user experience metrics

I monitor cache hit ratio (target: >90%), average response time (target: <50ms), and bandwidth savings. I use CDN analytics and application monitoring tools to track these metrics."

---

## Summary

CDN provides:
- **Faster content delivery** (reduced latency)
- **Reduced server load** (offloads traffic)
- **Better availability** (multiple edge servers)
- **Cost savings** (reduced bandwidth)
- **Global reach** (consistent performance)

**Use CDN for:**
- Static assets (CSS, JS, images)
- Product images
- File downloads
- Video content
- Fonts
- API response caching (with care)

**Best Practices:**
- Set appropriate cache headers
- Use versioning for cache busting
- Optimize images (WebP, multiple sizes)
- Implement fallback strategies
- Monitor CDN performance
- Secure private content with signed URLs

**Next Steps:**
- Implement CDN for product images
- Set up CloudFront or Cloudflare
- Optimize static assets
- Monitor CDN performance
- Implement cache invalidation strategy

