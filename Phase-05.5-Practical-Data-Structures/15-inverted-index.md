# 🎯 Inverted Index

## 0️⃣ Prerequisites

Before diving into Inverted Indexes, you need to understand:

### Forward Index
A **forward index** maps documents to their terms:
```
Document 1 → ["the", "quick", "brown", "fox"]
Document 2 → ["the", "lazy", "dog"]
Document 3 → ["quick", "brown", "dog"]
```

### Tokenization
**Tokenization** splits text into individual terms:
```
"The Quick Brown Fox" → ["the", "quick", "brown", "fox"]
```

### Hash Maps
**Hash maps** provide O(1) lookup by key. Inverted indexes are essentially specialized hash maps.

### Set Operations
**Intersection**, **union**, and **difference** of sets are used to combine search results:
```
Set A = {1, 2, 3}
Set B = {2, 3, 4}
A ∩ B = {2, 3}  // AND
A ∪ B = {1, 2, 3, 4}  // OR
```

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Core Problem: Finding Documents by Content

Imagine you have 1 billion web pages and need to find all pages containing "distributed systems".

**Naive approach (forward index scan):**
```java
List<Document> search(String term) {
    List<Document> results = new ArrayList<>();
    for (Document doc : allDocuments) {  // 1 billion documents!
        if (doc.contains(term)) {
            results.add(doc);
        }
    }
    return results;
}
// O(n × m) where n = documents, m = avg document length
// For 1 billion docs: HOURS per query!
```

### The Inverted Index Solution

Instead of document → terms, store terms → documents:

```
Forward Index:                    Inverted Index:
Doc1 → [quick, brown, fox]       quick → [Doc1, Doc3]
Doc2 → [lazy, dog]               brown → [Doc1, Doc3]
Doc3 → [quick, brown, dog]       fox   → [Doc1]
                                  lazy  → [Doc2]
                                  dog   → [Doc2, Doc3]
```

**Search for "quick":**
- Look up "quick" in inverted index
- Return [Doc1, Doc3]
- O(1) lookup + O(k) where k = matching documents

### Real-World Pain Points

**Scenario 1: Google Search**
```
Query: "distributed systems tutorial"
- Look up each term in inverted index
- Intersect posting lists
- Rank by relevance
- Return top 10 results

All in < 200ms for billions of documents!
```

**Scenario 2: E-commerce Product Search**
```
Query: "red running shoes size 10"
- Find products matching each term
- Filter by attributes
- Sort by relevance/price/rating
```

**Scenario 3: Log Search (Elasticsearch)**
```
Query: "error AND service:auth AND timestamp:[2024-01-01 TO 2024-01-31]"
- Find log entries matching criteria
- Aggregate by time, service, error type
```

### What Breaks Without Inverted Indexes?

| Without Inverted Index | With Inverted Index |
|------------------------|---------------------|
| O(n × m) per query | O(1) lookup + O(k) results |
| Minutes to hours | Milliseconds |
| Can't scale beyond small datasets | Handles billions of documents |
| No relevance ranking | TF-IDF, BM25 ranking |

---

## 2️⃣ Intuition and Mental Model

### The Book Index Analogy

A book's index at the back is an inverted index:

```
Book Index:
algorithm ............ 15, 42, 89, 156
binary search ........ 23, 45
data structure ....... 12, 34, 67, 89
hash table ........... 56, 78

To find pages about "algorithm":
1. Look up "algorithm" in index
2. Go to pages 15, 42, 89, 156

You don't scan the entire book!
```

### The Key Insight

```
┌─────────────────────────────────────────────────────────────────┐
│                 INVERTED INDEX KEY INSIGHT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Invert the relationship: term → documents"                    │
│                                                                  │
│  Components:                                                    │
│  1. Dictionary: All unique terms (vocabulary)                   │
│  2. Posting List: For each term, list of documents containing   │
│     it, often with positions and frequencies                    │
│                                                                  │
│  Query Processing:                                              │
│  - Single term: Return posting list                             │
│  - AND: Intersect posting lists                                 │
│  - OR: Union posting lists                                      │
│  - Phrase: Check positions in posting lists                     │
│                                                                  │
│  Preprocessing:                                                 │
│  - Tokenization: Split text into terms                          │
│  - Normalization: Lowercase, remove punctuation                 │
│  - Stemming: "running" → "run"                                  │
│  - Stop words: Remove "the", "a", "is"                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    INVERTED INDEX STRUCTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Dictionary (Term → Posting List Pointer)                       │
│  ┌──────────┬───────────────────────────────────────────────┐  │
│  │  Term    │  Posting List                                 │  │
│  ├──────────┼───────────────────────────────────────────────┤  │
│  │  apple   │  → [1, 5, 23, 45, 67]                         │  │
│  │  banana  │  → [2, 5, 12]                                 │  │
│  │  cherry  │  → [1, 2, 3, 4, 5, 6, 7, 8, ...]             │  │
│  │  ...     │  → ...                                        │  │
│  └──────────┴───────────────────────────────────────────────┘  │
│                                                                  │
│  Posting List Entry (can include):                              │
│  - Document ID                                                  │
│  - Term Frequency (TF): How many times term appears in doc     │
│  - Positions: Where in document term appears                   │
│  - Payloads: Custom data (e.g., field boost)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Posting List Formats

**Simple Posting List:**
```
term → [docId1, docId2, docId3, ...]
"apple" → [1, 5, 23, 45, 67]
```

**Posting List with Frequency:**
```
term → [(docId, tf), (docId, tf), ...]
"apple" → [(1, 3), (5, 1), (23, 2), (45, 5), (67, 1)]
          doc 1 has "apple" 3 times
```

**Posting List with Positions:**
```
term → [(docId, tf, [positions]), ...]
"apple" → [(1, 3, [5, 12, 45]), (5, 1, [8]), ...]
          doc 1 has "apple" at positions 5, 12, 45
```

### Index Construction

```
Algorithm: Build Inverted Index

1. For each document:
   a. Tokenize text into terms
   b. Normalize terms (lowercase, stem, etc.)
   c. For each term:
      - Add (docId, position) to term's posting list

2. Sort posting lists by docId (for efficient intersection)

3. Compress posting lists (delta encoding, variable-byte)

4. Build dictionary (term → posting list location)
```

### Query Processing

**Single Term Query:**
```
Query: "apple"
1. Look up "apple" in dictionary
2. Return posting list [1, 5, 23, 45, 67]
```

**AND Query (Intersection):**
```
Query: "apple AND banana"
1. Get posting list for "apple": [1, 5, 23, 45, 67]
2. Get posting list for "banana": [2, 5, 12]
3. Intersect: [5]  (only doc 5 has both)
```

**OR Query (Union):**
```
Query: "apple OR banana"
1. Get posting list for "apple": [1, 5, 23, 45, 67]
2. Get posting list for "banana": [2, 5, 12]
3. Union: [1, 2, 5, 12, 23, 45, 67]
```

**Phrase Query:**
```
Query: "quick brown fox"
1. Get posting lists with positions:
   "quick" → [(1, [5]), (3, [2])]
   "brown" → [(1, [6]), (3, [8])]
   "fox"   → [(1, [7])]
2. Find docs where positions are consecutive:
   Doc 1: quick@5, brown@6, fox@7 → consecutive! ✓
   Doc 3: quick@2, brown@8 → not consecutive ✗
3. Return [1]
```

---

## 4️⃣ Simulation: Step-by-Step Walkthrough

### Building an Inverted Index

**Documents:**
```
Doc 1: "The quick brown fox jumps"
Doc 2: "The lazy dog sleeps"
Doc 3: "Quick brown dogs are lazy"
```

**Step 1: Tokenize and Normalize**
```
Doc 1: ["the", "quick", "brown", "fox", "jumps"]
Doc 2: ["the", "lazy", "dog", "sleeps"]
Doc 3: ["quick", "brown", "dogs", "are", "lazy"]

After stemming (dogs → dog, jumps → jump, sleeps → sleep):
Doc 1: ["the", "quick", "brown", "fox", "jump"]
Doc 2: ["the", "lazy", "dog", "sleep"]
Doc 3: ["quick", "brown", "dog", "are", "lazy"]

After removing stop words (the, are):
Doc 1: ["quick", "brown", "fox", "jump"]
Doc 2: ["lazy", "dog", "sleep"]
Doc 3: ["quick", "brown", "dog", "lazy"]
```

**Step 2: Build Posting Lists**
```
Processing Doc 1:
  quick → [1]
  brown → [1]
  fox   → [1]
  jump  → [1]

Processing Doc 2:
  lazy  → [2]
  dog   → [2]
  sleep → [2]

Processing Doc 3:
  quick → [1, 3]
  brown → [1, 3]
  dog   → [2, 3]
  lazy  → [2, 3]
```

**Final Inverted Index:**
```
brown → [1, 3]
dog   → [2, 3]
fox   → [1]
jump  → [1]
lazy  → [2, 3]
quick → [1, 3]
sleep → [2]
```

### Query Execution

**Query: "quick AND lazy"**
```
1. Look up "quick": [1, 3]
2. Look up "lazy": [2, 3]
3. Intersect using merge algorithm:
   
   Pointer A: [1, 3]  → 1
   Pointer B: [2, 3]  → 2
   
   1 < 2: advance A → 3
   3 > 2: advance B → 3
   3 = 3: match! Add 3, advance both
   
   Result: [3]

4. Return Doc 3
```

**Query: "brown dog"** (phrase)
```
With positions:
brown → [(1, [2]), (3, [1])]  // doc 1 pos 2, doc 3 pos 1
dog   → [(2, [1]), (3, [2])]  // doc 2 pos 1, doc 3 pos 2

Check Doc 3:
  brown @ position 1
  dog @ position 2
  2 - 1 = 1 → consecutive! ✓

Result: [3]
```

---

## 5️⃣ How Engineers Use This in Production

### Elasticsearch

Elasticsearch is built on Apache Lucene, which uses inverted indexes:

```json
// Index a document
PUT /products/_doc/1
{
  "name": "Red Running Shoes",
  "description": "Lightweight running shoes for marathon training",
  "price": 129.99,
  "category": "footwear"
}

// Search
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "running shoes" } }
      ],
      "filter": [
        { "term": { "category": "footwear" } },
        { "range": { "price": { "lte": 150 } } }
      ]
    }
  }
}
```

**Elasticsearch Index Structure:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    ELASTICSEARCH INDEX                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Index: products                                                │
│  ├── Shard 0                                                    │
│  │   ├── Segment 0 (immutable)                                 │
│  │   │   ├── Inverted Index (terms → docs)                     │
│  │   │   ├── Doc Values (docs → field values, for sorting)     │
│  │   │   └── Stored Fields (original JSON)                     │
│  │   └── Segment 1                                             │
│  ├── Shard 1                                                    │
│  └── Shard 2                                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Apache Lucene

Lucene is the underlying library for Elasticsearch and Solr:

```java
// Create index
Directory directory = FSDirectory.open(Paths.get("/tmp/index"));
IndexWriterConfig config = new IndexWriterConfig(new StandardAnalyzer());
IndexWriter writer = new IndexWriter(directory, config);

// Add document
Document doc = new Document();
doc.add(new TextField("title", "Quick Brown Fox", Field.Store.YES));
doc.add(new TextField("content", "The quick brown fox jumps", Field.Store.YES));
writer.addDocument(doc);
writer.close();

// Search
IndexReader reader = DirectoryReader.open(directory);
IndexSearcher searcher = new IndexSearcher(reader);

Query query = new TermQuery(new Term("content", "quick"));
TopDocs results = searcher.search(query, 10);

for (ScoreDoc scoreDoc : results.scoreDocs) {
    Document d = searcher.doc(scoreDoc.doc);
    System.out.println(d.get("title") + " - Score: " + scoreDoc.score);
}
```

### PostgreSQL Full-Text Search

PostgreSQL has built-in full-text search with inverted indexes (GIN):

```sql
-- Create table with tsvector column
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    body TEXT,
    search_vector TSVECTOR
);

-- Create GIN index (inverted index)
CREATE INDEX idx_search ON articles USING GIN(search_vector);

-- Update search vector
UPDATE articles SET search_vector = 
    setweight(to_tsvector('english', title), 'A') ||
    setweight(to_tsvector('english', body), 'B');

-- Search
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'distributed & systems') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

---

## 6️⃣ Implementation in Java

### Basic Inverted Index

```java
import java.util.*;
import java.util.regex.*;

/**
 * Simple inverted index implementation.
 */
public class InvertedIndex {
    
    // Term → List of (docId, positions)
    private final Map<String, List<Posting>> index;
    private final Map<Integer, String> documents;
    private int docIdCounter;
    
    /**
     * A posting represents a term occurrence in a document.
     */
    static class Posting implements Comparable<Posting> {
        int docId;
        int termFrequency;
        List<Integer> positions;
        
        Posting(int docId) {
            this.docId = docId;
            this.termFrequency = 0;
            this.positions = new ArrayList<>();
        }
        
        void addPosition(int position) {
            positions.add(position);
            termFrequency++;
        }
        
        @Override
        public int compareTo(Posting other) {
            return Integer.compare(this.docId, other.docId);
        }
    }
    
    public InvertedIndex() {
        this.index = new HashMap<>();
        this.documents = new HashMap<>();
        this.docIdCounter = 0;
    }
    
    /**
     * Adds a document to the index.
     */
    public int addDocument(String content) {
        int docId = docIdCounter++;
        documents.put(docId, content);
        
        // Tokenize and index
        List<String> tokens = tokenize(content);
        Map<String, Posting> docPostings = new HashMap<>();
        
        for (int position = 0; position < tokens.size(); position++) {
            String term = normalize(tokens.get(position));
            if (term.isEmpty() || isStopWord(term)) continue;
            
            docPostings.computeIfAbsent(term, k -> new Posting(docId))
                       .addPosition(position);
        }
        
        // Merge into main index
        for (Map.Entry<String, Posting> entry : docPostings.entrySet()) {
            index.computeIfAbsent(entry.getKey(), k -> new ArrayList<>())
                 .add(entry.getValue());
        }
        
        return docId;
    }
    
    /**
     * Searches for documents containing the term.
     */
    public List<Integer> search(String term) {
        term = normalize(term);
        List<Posting> postings = index.get(term);
        if (postings == null) return Collections.emptyList();
        
        List<Integer> result = new ArrayList<>();
        for (Posting p : postings) {
            result.add(p.docId);
        }
        return result;
    }
    
    /**
     * AND query: documents containing all terms.
     */
    public List<Integer> searchAnd(String... terms) {
        if (terms.length == 0) return Collections.emptyList();
        
        // Get posting lists
        List<List<Integer>> postingLists = new ArrayList<>();
        for (String term : terms) {
            List<Integer> docs = search(term);
            if (docs.isEmpty()) return Collections.emptyList();
            postingLists.add(docs);
        }
        
        // Sort by size (smallest first for efficiency)
        postingLists.sort(Comparator.comparingInt(List::size));
        
        // Intersect
        Set<Integer> result = new HashSet<>(postingLists.get(0));
        for (int i = 1; i < postingLists.size(); i++) {
            result.retainAll(new HashSet<>(postingLists.get(i)));
        }
        
        return new ArrayList<>(result);
    }
    
    /**
     * OR query: documents containing any term.
     */
    public List<Integer> searchOr(String... terms) {
        Set<Integer> result = new HashSet<>();
        for (String term : terms) {
            result.addAll(search(term));
        }
        return new ArrayList<>(result);
    }
    
    /**
     * Phrase query: documents containing exact phrase.
     */
    public List<Integer> searchPhrase(String phrase) {
        List<String> terms = tokenize(phrase);
        if (terms.isEmpty()) return Collections.emptyList();
        
        // Normalize terms
        List<String> normalizedTerms = new ArrayList<>();
        for (String term : terms) {
            String normalized = normalize(term);
            if (!normalized.isEmpty() && !isStopWord(normalized)) {
                normalizedTerms.add(normalized);
            }
        }
        
        if (normalizedTerms.isEmpty()) return Collections.emptyList();
        
        // Get first term's postings
        List<Posting> firstPostings = index.get(normalizedTerms.get(0));
        if (firstPostings == null) return Collections.emptyList();
        
        List<Integer> result = new ArrayList<>();
        
        for (Posting posting : firstPostings) {
            if (containsPhrase(posting.docId, normalizedTerms)) {
                result.add(posting.docId);
            }
        }
        
        return result;
    }
    
    private boolean containsPhrase(int docId, List<String> terms) {
        // Get positions for first term
        List<Posting> firstPostings = index.get(terms.get(0));
        Posting firstPosting = null;
        for (Posting p : firstPostings) {
            if (p.docId == docId) {
                firstPosting = p;
                break;
            }
        }
        if (firstPosting == null) return false;
        
        // For each position of first term
        for (int startPos : firstPosting.positions) {
            boolean match = true;
            
            // Check if subsequent terms are at consecutive positions
            for (int i = 1; i < terms.size(); i++) {
                int expectedPos = startPos + i;
                List<Posting> termPostings = index.get(terms.get(i));
                
                if (termPostings == null) {
                    match = false;
                    break;
                }
                
                boolean found = false;
                for (Posting p : termPostings) {
                    if (p.docId == docId && p.positions.contains(expectedPos)) {
                        found = true;
                        break;
                    }
                }
                
                if (!found) {
                    match = false;
                    break;
                }
            }
            
            if (match) return true;
        }
        
        return false;
    }
    
    /**
     * Tokenizes text into terms.
     */
    private List<String> tokenize(String text) {
        List<String> tokens = new ArrayList<>();
        Matcher matcher = Pattern.compile("\\w+").matcher(text);
        while (matcher.find()) {
            tokens.add(matcher.group());
        }
        return tokens;
    }
    
    /**
     * Normalizes a term (lowercase, basic stemming).
     */
    private String normalize(String term) {
        term = term.toLowerCase();
        // Very basic stemming
        if (term.endsWith("ing")) {
            term = term.substring(0, term.length() - 3);
        } else if (term.endsWith("ed")) {
            term = term.substring(0, term.length() - 2);
        } else if (term.endsWith("s") && term.length() > 3) {
            term = term.substring(0, term.length() - 1);
        }
        return term;
    }
    
    /**
     * Checks if term is a stop word.
     */
    private boolean isStopWord(String term) {
        Set<String> stopWords = Set.of(
            "the", "a", "an", "and", "or", "but", "in", "on", "at",
            "to", "for", "of", "with", "by", "is", "are", "was", "were"
        );
        return stopWords.contains(term);
    }
    
    /**
     * Gets document content by ID.
     */
    public String getDocument(int docId) {
        return documents.get(docId);
    }
    
    /**
     * Gets index statistics.
     */
    public void printStats() {
        System.out.println("Documents: " + documents.size());
        System.out.println("Unique terms: " + index.size());
        
        int totalPostings = 0;
        for (List<Posting> postings : index.values()) {
            totalPostings += postings.size();
        }
        System.out.println("Total postings: " + totalPostings);
    }
}
```

### TF-IDF Scoring

```java
import java.util.*;

/**
 * TF-IDF scoring for search results.
 */
public class TfIdfScorer {
    
    private final InvertedIndex index;
    private final int totalDocuments;
    private final Map<String, Integer> documentFrequency;
    
    public TfIdfScorer(InvertedIndex index, int totalDocuments) {
        this.index = index;
        this.totalDocuments = totalDocuments;
        this.documentFrequency = new HashMap<>();
    }
    
    /**
     * Calculates TF-IDF score for a term in a document.
     * 
     * TF (Term Frequency) = count of term in doc / total terms in doc
     * IDF (Inverse Document Frequency) = log(total docs / docs containing term)
     * TF-IDF = TF × IDF
     */
    public double score(String term, int docId, int termFrequency, int docLength) {
        // Term Frequency (normalized)
        double tf = (double) termFrequency / docLength;
        
        // Inverse Document Frequency
        int df = getDocumentFrequency(term);
        double idf = Math.log((double) totalDocuments / (df + 1)) + 1;
        
        return tf * idf;
    }
    
    /**
     * Scores and ranks search results.
     */
    public List<ScoredDocument> searchAndRank(String query, int topK) {
        String[] terms = query.toLowerCase().split("\\s+");
        Map<Integer, Double> scores = new HashMap<>();
        
        for (String term : terms) {
            List<Integer> docs = index.search(term);
            for (int docId : docs) {
                // Simplified: assume tf=1, docLength=100
                double termScore = score(term, docId, 1, 100);
                scores.merge(docId, termScore, Double::sum);
            }
        }
        
        // Sort by score
        List<ScoredDocument> results = new ArrayList<>();
        for (Map.Entry<Integer, Double> entry : scores.entrySet()) {
            results.add(new ScoredDocument(entry.getKey(), entry.getValue()));
        }
        results.sort((a, b) -> Double.compare(b.score, a.score));
        
        return results.subList(0, Math.min(topK, results.size()));
    }
    
    private int getDocumentFrequency(String term) {
        return documentFrequency.getOrDefault(term, 1);
    }
    
    static class ScoredDocument {
        int docId;
        double score;
        
        ScoredDocument(int docId, double score) {
            this.docId = docId;
            this.score = score;
        }
        
        @Override
        public String toString() {
            return String.format("Doc %d (score: %.4f)", docId, score);
        }
    }
}
```

### Testing the Implementation

```java
public class InvertedIndexTest {
    
    public static void main(String[] args) {
        testBasicSearch();
        testBooleanQueries();
        testPhraseQuery();
    }
    
    static void testBasicSearch() {
        System.out.println("=== Basic Search ===");
        
        InvertedIndex index = new InvertedIndex();
        
        index.addDocument("The quick brown fox jumps over the lazy dog");
        index.addDocument("A quick brown dog outpaces a lazy fox");
        index.addDocument("The dog is not lazy but the fox is quick");
        
        index.printStats();
        
        System.out.println("\nSearch 'quick': " + index.search("quick"));
        System.out.println("Search 'lazy': " + index.search("lazy"));
        System.out.println("Search 'cat': " + index.search("cat"));
    }
    
    static void testBooleanQueries() {
        System.out.println("\n=== Boolean Queries ===");
        
        InvertedIndex index = new InvertedIndex();
        
        index.addDocument("Java programming language");
        index.addDocument("Python programming language");
        index.addDocument("Java virtual machine");
        index.addDocument("Python machine learning");
        
        System.out.println("'Java' AND 'programming': " + 
            index.searchAnd("Java", "programming"));
        System.out.println("'Java' OR 'Python': " + 
            index.searchOr("Java", "Python"));
        System.out.println("'machine' AND 'learning': " + 
            index.searchAnd("machine", "learning"));
    }
    
    static void testPhraseQuery() {
        System.out.println("\n=== Phrase Query ===");
        
        InvertedIndex index = new InvertedIndex();
        
        index.addDocument("The quick brown fox jumps");
        index.addDocument("A brown quick fox runs");
        index.addDocument("Quick brown foxes are fast");
        
        System.out.println("Phrase 'quick brown': " + 
            index.searchPhrase("quick brown"));
        System.out.println("Phrase 'brown quick': " + 
            index.searchPhrase("brown quick"));
    }
}
```

---

## 7️⃣ Index Compression

### Why Compress?

Inverted indexes can be huge:
- 1 billion documents
- 100,000 unique terms
- Average 1000 postings per term
- = 100 billion integers = 400 GB uncompressed!

### Delta Encoding

Store differences instead of absolute values:

```
Original: [1, 5, 23, 45, 67, 89]
Deltas:   [1, 4, 18, 22, 22, 22]
          (first value, then differences)

Deltas are smaller numbers → compress better
```

### Variable-Byte Encoding

Use fewer bytes for smaller numbers:

```
Number    Binary           Variable-Byte
1         00000001         1 byte:  00000001
127       01111111         1 byte:  01111111
128       10000000         2 bytes: 10000001 00000000
16383     11111111111111   2 bytes: 11111111 01111111
```

### Posting List Compression Example

```java
/**
 * Variable-byte encoding for posting lists.
 */
public class VByteEncoder {
    
    public static byte[] encode(int[] numbers) {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        
        int prev = 0;
        for (int num : numbers) {
            int delta = num - prev;
            prev = num;
            
            // Encode delta using variable bytes
            while (delta >= 128) {
                out.write((delta & 0x7F) | 0x80);
                delta >>>= 7;
            }
            out.write(delta);
        }
        
        return out.toByteArray();
    }
    
    public static int[] decode(byte[] bytes) {
        List<Integer> numbers = new ArrayList<>();
        int current = 0;
        int shift = 0;
        int prev = 0;
        
        for (byte b : bytes) {
            if ((b & 0x80) != 0) {
                current |= (b & 0x7F) << shift;
                shift += 7;
            } else {
                current |= b << shift;
                prev += current;
                numbers.add(prev);
                current = 0;
                shift = 0;
            }
        }
        
        return numbers.stream().mapToInt(i -> i).toArray();
    }
}
```

---

## 8️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Aspect | Inverted Index | B+ Tree | Full Scan |
|--------|----------------|---------|-----------|
| Text search | Excellent | Poor | Works |
| Exact match | Good | Excellent | Works |
| Range query | Not supported | Excellent | Works |
| Index size | Large | Moderate | None |
| Update cost | High | Moderate | None |

### Common Pitfalls

**1. Not handling stop words**

```java
// BAD: Indexing common words
index.add("the");  // Appears in every document!
// Huge posting list, no search value

// GOOD: Filter stop words
if (!stopWords.contains(term)) {
    index.add(term);
}
```

**2. Ignoring stemming/lemmatization**

```java
// BAD: Exact terms only
// "running" and "run" are different terms
// User searching "run" misses "running" documents

// GOOD: Stem terms
String stemmed = stemmer.stem(term);  // "running" → "run"
index.add(stemmed);
```

**3. Not compressing posting lists**

```java
// BAD: Storing raw document IDs
// [1, 5, 23, 45, 67, 89, 123, ...]
// 4 bytes per ID × millions of documents = huge!

// GOOD: Delta encoding + variable-byte
// [1, 4, 18, 22, 22, 22, 34, ...]
// Smaller numbers, better compression
```

**4. Inefficient posting list intersection**

```java
// BAD: Nested loops for AND query
for (int docA : postingListA) {
    for (int docB : postingListB) {  // O(n²)!
        if (docA == docB) results.add(docA);
    }
}

// GOOD: Merge algorithm (sorted lists)
// O(n + m) with two pointers
```

### Performance Gotchas

**1. Index update latency**

```java
// Problem: Real-time indexing is expensive
// Solution: Batch updates, near-real-time refresh
// Elasticsearch: 1 second refresh interval by default
```

**2. Memory pressure**

```java
// Problem: Large dictionaries in memory
// Solution: Memory-mapped files, tiered storage
```

**3. Query complexity**

```java
// Problem: Complex queries (many ANDs/ORs) are slow
// Solution: Query optimization, early termination
// Process smallest posting list first
```

---

## 9️⃣ When NOT to Use Inverted Indexes

### Anti-Patterns

**1. Exact key-value lookups**

```java
// WRONG: Inverted index for user lookup by ID
// Tokenization overhead, unnecessary

// RIGHT: Use B+ Tree or hash index
```

**2. Numeric range queries**

```java
// WRONG: Inverted index for "price between 10 and 50"
// Each price is a separate term

// RIGHT: Use B+ Tree with range scan
// Or numeric range encoding in Elasticsearch
```

**3. Frequently updated documents**

```java
// WRONG: Inverted index for real-time analytics
// Index updates are expensive

// RIGHT: Use column store or time-series DB
```

**4. Small datasets**

```java
// WRONG: Inverted index for 100 documents
// Overhead not worth it

// RIGHT: Simple linear scan or in-memory filter
```

### Better Alternatives

| Use Case | Better Alternative |
|----------|-------------------|
| Key-value lookup | Hash index, B+ Tree |
| Numeric ranges | B+ Tree |
| Real-time analytics | Column store |
| Similarity search | Vector index |
| Exact string match | Hash index |

---

## 🔟 Interview Follow-Up Questions with Answers

### L4 (Entry-Level) Questions

**Q1: What is an inverted index and why is it used for search?**

**Answer**: An inverted index maps terms to the documents containing them (opposite of a forward index which maps documents to terms). It's used for search because:
1. **Fast lookup**: Finding documents with a term is O(1) dictionary lookup
2. **Boolean queries**: AND/OR operations are set intersections/unions
3. **Scalability**: Works for billions of documents
4. **Ranking**: Can store term frequencies for relevance scoring

Example: Instead of scanning all documents for "cat", look up "cat" in the index and immediately get [doc1, doc5, doc23].

**Q2: What is TF-IDF and how does it work?**

**Answer**: TF-IDF (Term Frequency - Inverse Document Frequency) is a scoring method for search relevance.

- **TF (Term Frequency)**: How often a term appears in a document. Higher = more relevant.
- **IDF (Inverse Document Frequency)**: How rare a term is across all documents. Rare terms are more discriminative.
- **TF-IDF = TF × IDF**

Example:
- "the" appears in every document → low IDF → low score
- "quantum" appears in few documents → high IDF → high score
- A document with "quantum" 5 times scores higher than one with "quantum" once

### L5 (Senior) Questions

**Q3: How do you handle phrase queries in an inverted index?**

**Answer**: Phrase queries require position information in posting lists:

1. **Store positions**: Each posting includes term positions in the document
   ```
   "quick" → [(doc1, [5, 23]), (doc3, [2])]
   "brown" → [(doc1, [6, 24]), (doc3, [8])]
   ```

2. **Query processing**:
   - Get posting lists for all terms
   - Find documents containing all terms
   - Check if positions are consecutive

3. **Optimization**:
   - Next-word index: Store pairs of consecutive words
   - Biword index: "quick brown" as a single term
   - Positional index with skip pointers

**Q4: How would you design a search system for 1 billion documents?**

**Answer**:

```
Architecture:

1. Indexing Pipeline:
   - Crawlers fetch documents
   - Parser extracts text, metadata
   - Tokenizer splits into terms
   - Indexer builds inverted index
   - Distributed across shards

2. Index Storage:
   - Shard by document ID (random distribution)
   - Each shard has complete inverted index for its docs
   - Replicas for availability

3. Query Processing:
   - Query parser analyzes query
   - Scatter query to all shards
   - Each shard returns top-K results
   - Merge and re-rank globally

4. Optimizations:
   - Tiered index: Hot/warm/cold
   - Caching: Query cache, posting list cache
   - Early termination: Stop after enough good results
   - Skip pointers in posting lists

5. Relevance:
   - BM25 or TF-IDF base scoring
   - PageRank for authority
   - Click-through rate feedback
   - Machine learning re-ranking
```

### L6 (Staff) Questions

**Q5: Design Elasticsearch-like search infrastructure for a large e-commerce platform.**

**Answer**:

```
Architecture:

1. Data Model:
   - Products: name, description, category, price, attributes
   - Multiple indexes: products, reviews, sellers
   - Denormalized for search performance

2. Index Design:
   - Shard by category (query locality)
   - Replicas: 2 per shard
   - Separate indexes for different query patterns

3. Ingestion:
   - Kafka for product updates
   - Bulk indexing for efficiency
   - Near-real-time refresh (1 second)

4. Query Flow:
   - Parse query, extract filters
   - Route to relevant shards
   - Execute: filters first, then full-text
   - Aggregate facets
   - Re-rank with ML model
   - Return top results

5. Features:
   - Autocomplete: Edge n-grams, completion suggester
   - Faceted search: Aggregations on category, price range
   - Typo tolerance: Fuzzy matching, phonetic analysis
   - Synonyms: "shoes" = "footwear"
   - Personalization: User history boost

6. Operations:
   - Blue-green deployments for index updates
   - Monitoring: Query latency, indexing lag
   - Capacity planning: Queries/sec, index size
```

---

## 1️⃣1️⃣ One Clean Mental Summary

An inverted index maps terms to the documents containing them, enabling O(1) lookup for any term. Each term has a posting list: a sorted list of document IDs (plus optional frequencies and positions). Boolean queries (AND/OR) become set operations on posting lists. Phrase queries use position information to verify term adjacency. TF-IDF or BM25 score relevance by balancing term frequency (how often in this doc) with inverse document frequency (how rare across all docs). Compression (delta encoding, variable-byte) reduces index size dramatically. Inverted indexes power search engines like Google, Elasticsearch, and full-text search in databases.

---

## Summary

Inverted indexes are essential for:
- **Search engines**: Google, Bing, DuckDuckGo
- **Enterprise search**: Elasticsearch, Solr, Splunk
- **Database full-text search**: PostgreSQL, MySQL, MongoDB
- **Log analysis**: ELK stack, Splunk

Key takeaways:
1. Term → Document mapping (inverse of forward index)
2. Posting lists store document IDs, frequencies, positions
3. Boolean queries = set operations (intersect, union)
4. Phrase queries need position information
5. TF-IDF/BM25 for relevance ranking
6. Compression is critical for scale (delta + variable-byte)
7. Sharding for horizontal scaling

