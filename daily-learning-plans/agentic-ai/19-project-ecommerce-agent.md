# Agentic AI — Day 19: Project — E-Commerce Shopping Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Build a Conversational Shopping Agent — Search, Recommend, Compare, and Checkout**

This agent powers an e-commerce chatbot: users describe what they want in natural language, the agent searches the product catalog (using RAG + structured queries), compares options, provides personalised recommendations, and guides users through checkout. It's a multi-tool agent with memory that remembers preferences across the conversation.

---

## 2. Architecture

```
Customer: "I need running shoes under $150, good for flat feet"
       │
       ▼
┌──────────────────┐
│ Shopping Agent   │
│ (Spring AI)      │
└───┬──────────────┘
    │
    ├── Tool: search_products(query, filters)
    ├── Tool: get_product_details(product_id)
    ├── Tool: compare_products(product_ids)
    ├── Tool: check_inventory(product_id, size)
    ├── Tool: add_to_cart(user_id, product_id, size, quantity)
    ├── Tool: get_cart(user_id)
    │
    ▼
Memory: remembers preferences (flat feet, budget $150, running)
```

---

## 3. Java Implementation (Spring AI)

### 3.1 Tools

```java
@Component
@RequiredArgsConstructor
public class ShoppingTools {

    private final ProductRepository productRepo;
    private final CartService cartService;
    private final VectorStore productVectorStore;

    @Tool("Search products by natural language query with optional filters. "
        + "Returns matching products with name, price, rating, and key features.")
    public List<ProductSummary> searchProducts(
            @ToolParam("Natural language search query (e.g., 'running shoes for flat feet')") String query,
            @ToolParam(value = "Maximum price filter", required = false) Double maxPrice,
            @ToolParam(value = "Minimum rating (1-5)", required = false) Double minRating,
            @ToolParam(value = "Category filter", required = false) String category) {

        // Semantic search via vector store
        SearchRequest searchReq = SearchRequest.builder()
            .query(query)
            .topK(10)
            .similarityThreshold(0.6)
            .build();

        List<Document> docs = productVectorStore.similaritySearch(searchReq);
        List<Long> productIds = docs.stream()
            .map(d -> Long.parseLong(d.getMetadata().get("productId").toString()))
            .toList();

        // Apply structured filters
        return productRepo.findByIdsWithFilters(productIds, maxPrice, minRating, category)
            .stream()
            .map(p -> new ProductSummary(p.getId(), p.getName(), p.getPrice(),
                p.getRating(), p.getKeyFeatures().subList(0, Math.min(3, p.getKeyFeatures().size()))))
            .toList();
    }

    @Tool("Get detailed product information including full description, all features, "
        + "available sizes, reviews summary, and similar products.")
    public ProductDetails getProductDetails(
            @ToolParam("Product ID") Long productId) {
        Product p = productRepo.findById(productId).orElseThrow();
        List<String> reviews = reviewRepo.findTopByProductId(productId, 3)
            .stream().map(r -> r.getRating() + "★: " + r.getSummary()).toList();
        return new ProductDetails(p.getId(), p.getName(), p.getDescription(),
            p.getPrice(), p.getRating(), p.getFeatures(), p.getAvailableSizes(), reviews);
    }

    @Tool("Compare 2-4 products side by side on price, rating, key features, and pros/cons.")
    public String compareProducts(
            @ToolParam("List of product IDs to compare (2-4 products)") List<Long> productIds) {
        List<Product> products = productRepo.findAllById(productIds);
        StringBuilder sb = new StringBuilder("| Feature |");
        for (Product p : products) sb.append(" ").append(p.getName()).append(" |");
        sb.append("\n|---------|");
        for (int i = 0; i < products.size(); i++) sb.append("--------|");
        sb.append("\n| Price |");
        for (Product p : products) sb.append(" $").append(p.getPrice()).append(" |");
        sb.append("\n| Rating |");
        for (Product p : products) sb.append(" ").append(p.getRating()).append("★ |");
        sb.append("\n| Key Feature |");
        for (Product p : products) sb.append(" ").append(p.getKeyFeatures().get(0)).append(" |");
        return sb.toString();
    }

    @Tool("Check if a product is available in a specific size. Returns stock status.")
    public String checkInventory(
            @ToolParam("Product ID") Long productId,
            @ToolParam("Size (e.g., '10', 'M', 'XL')") String size) {
        int stock = inventoryService.getStock(productId, size);
        if (stock == 0) return "OUT OF STOCK for size " + size;
        if (stock < 5) return "LOW STOCK: only " + stock + " left in size " + size;
        return "IN STOCK in size " + size;
    }

    @Tool("Add a product to the user's shopping cart.")
    public String addToCart(
            @ToolParam("User ID") Long userId,
            @ToolParam("Product ID") Long productId,
            @ToolParam("Size") String size,
            @ToolParam(value = "Quantity (default 1)", required = false) Integer quantity) {
        int qty = quantity != null ? quantity : 1;
        cartService.addItem(userId, productId, size, qty);
        Cart cart = cartService.getCart(userId);
        return String.format("Added to cart! Cart now has %d items, total: $%.2f",
            cart.getItemCount(), cart.getTotal());
    }

    @Tool("Get the current shopping cart contents for a user.")
    public String getCart(@ToolParam("User ID") Long userId) {
        Cart cart = cartService.getCart(userId);
        if (cart.isEmpty()) return "Your cart is empty.";
        StringBuilder sb = new StringBuilder("Your cart:\n");
        for (CartItem item : cart.getItems()) {
            sb.append(String.format("- %s (Size: %s, Qty: %d) — $%.2f\n",
                item.getProductName(), item.getSize(), item.getQuantity(), item.getSubtotal()));
        }
        sb.append(String.format("\nTotal: $%.2f", cart.getTotal()));
        return sb.toString();
    }
}
```

### 3.2 Agent Configuration

```java
@Bean
public ChatClient shoppingAgent(ChatClient.Builder builder) {
    return builder
        .defaultSystem("""
            You are a friendly, knowledgeable shopping assistant for an online store.

            Behaviour:
            - When a user describes what they want, search for products matching their needs.
            - Recommend 2-3 options, explaining WHY each fits their requirements.
            - If the user seems undecided, offer to compare products side by side.
            - Remember their preferences (budget, size, style) throughout the conversation.
            - When they're ready, help them add items to cart and check inventory.
            - Be conversational and helpful, not pushy.

            Rules:
            - Never recommend out-of-stock items without mentioning the stock issue.
            - Always confirm size before adding to cart.
            - If budget is mentioned, respect it strictly.

            Current user ID: {userId}
            """)
        .defaultTools(shoppingTools)
        .build();
}
```

### 3.3 Controller with Memory

```java
@PostMapping("/v1/shop/chat")
public AgentResponse chat(@RequestBody ShopRequest req,
                           @AuthenticationPrincipal UserPrincipal user) {
    String response = shoppingAgent.prompt()
        .system(s -> s.param("userId", user.getId()))
        .user(req.message())
        .advisors(new MessageChatMemoryAdvisor(chatMemory, req.conversationId(), 30))
        .call()
        .content();

    return new AgentResponse(req.conversationId(), response);
}
```

---

## 4. Product Ingestion for Semantic Search

```java
@Service
public class ProductIngestionService {

    public void ingestProducts(List<Product> products) {
        List<Document> docs = products.stream()
            .map(p -> {
                String content = p.getName() + ". " + p.getDescription() + ". "
                    + "Features: " + String.join(", ", p.getFeatures()) + ". "
                    + "Category: " + p.getCategory() + ". "
                    + "Price: $" + p.getPrice();
                return new Document(content, Map.of(
                    "productId", p.getId(),
                    "category", p.getCategory(),
                    "price", p.getPrice()
                ));
            })
            .toList();
        vectorStore.add(docs);
    }
}
```

---

## 5. Test Conversations

```
User: "I need running shoes under $150, I have flat feet"
Agent: [searches products with query + price filter]
       "Here are 3 great options for flat feet runners under $150:
        1. Brooks Beast 24 ($139.99, 4.5★) — Maximum stability, great arch support
        2. ASICS Gel-Kayano 31 ($149.99, 4.7★) — Premium cushioning with stability
        3. New Balance 860v14 ($129.99, 4.3★) — Best value, good for overpronation

        Would you like to compare any of these side by side?"

User: "Compare the first two"
Agent: [calls compare_products]
       "| Feature | Brooks Beast 24 | ASICS Gel-Kayano 31 |
        | Price   | $139.99         | $149.99             |
        ..."

User: "I'll go with the ASICS, size 10"
Agent: [checks inventory] "Great choice! Size 10 is in stock."
       [adds to cart] "Added to your cart. Total: $149.99. Ready to checkout?"
```

---

## 6. Hands-On Exercise

1. Create a Spring Boot project with Spring AI and mock product data (20 products).
2. Ingest products into a vector store (in-memory for dev).
3. Implement all 6 shopping tools.
4. Configure the agent with the system prompt above.
5. Test the full conversation flow: search → compare → add to cart.
6. Verify conversation memory: mention budget once, then later ask for recommendations without repeating it.

---

## 7. Key Takeaways

1. **Semantic search + structured filters = best product search** — combine vector similarity with price/rating filters.
2. **Memory is critical for shopping** — budget, size preferences, and conversation context must persist.
3. **Recommend, don't just list** — explain WHY each product matches the user's specific needs.
4. **Always check inventory before committing** — nothing worse than "sorry, out of stock" after checkout.

---

*Day 19. The best shopping assistant doesn't just show products — it understands what you need and why.*
