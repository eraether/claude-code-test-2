# LLM-Parsed Provenance-Grounded Knowledge Graphs

## The Core Insight

Understanding is not about words - words are arbitrary symbols. "Dog", "chien", "perro" are all different labels for the same concept. What matters is the *conceptual structure* underlying language.

Current approaches fail because they operate on linguistic tokens rather than extracting the concepts those tokens represent. But we can't extract concepts from raw text easily - the linguistic → conceptual mapping is incredibly complex.

**Solution:** LLMs already perform this mapping implicitly. Use the LLM not as a knowledge source (hallucination risk), but as a **parser** that converts natural language into explicit conceptual graph structures.

---

## Core Principles

### 1. Concepts are Possibility Spaces, Not Prototypes

A concept is not a single canonical example - it's the entire space of possible instantiations.

**CONCEPT_DOG contains:**
- Properties that can vary: {fur_color: [white, brown, black, spotted, ...], leg_count: [4, 3, 2, ...], size: [tiny, small, medium, large, ...]}
- Behaviors: {barks, doesn't_bark, rescues_people, fights_crime, ...}
- Instances: {lassie, snoopy, my_neighbor's_pug, fictional_superhero_dog, ...}
- Contexts: {real_world, fictional, hypothetical, ...}

**DOG_PROTOTYPE** is a derived concept - the typical or most common point in DOG-space:
- fur_color: brown
- leg_count: 4
- behaviors: barks
- size: medium

This model naturally handles:
- **Exceptions:** "The three-legged dog" - still in DOG-space
- **Fiction vs reality:** Superhero dogs exist in DOG-space, just in different context (fictional vs real)
- **Negation:** "The dog didn't bark" - locates this instance in the non-barking region of DOG-space
- **Evolution:** DOG-space expands as we encounter new properties ("can skateboard")

### 2. Every Fact Has Provenance

No fact exists in the knowledge graph without a source. Every node, edge, and property is grounded in specific text.

**Example:**
```
Source: "The dog wasn't barking" (Sherlock Holmes - Silver Blaze, line 247)
Parsed into:
  - CONCEPT: DOG (instance)
  - Property: barking = FALSE
  - Temporal context: past continuous
  - Provenance: [source_id: "sherlock_silver_blaze", line: 247, confidence: 0.95]
```

**Benefits:**
- Traceability: Can always answer "where did this come from?"
- Contradiction resolution: Two sources disagree → both are stored with provenance, not collapsed
- Confidence: Multiple sources supporting same claim increases confidence
- No hallucination: LLM doesn't generate facts, only parses provided text

### 3. Self-Consistency as the Organizing Principle

The graph's structure is optimized for internal logical consistency, not prediction accuracy.

**Consistency means:**
- Transitive relationships hold: If A is-a B, and B is-a C, then A is-a C
- Mutually exclusive properties are marked: Can't be "entirely black" AND "entirely white" simultaneously
- Constraints are respected: If DOG is-a MAMMAL, and MAMMAL requires warm-blooded, then DOG must be warm-blooded

**When new information creates inconsistency:**
1. Check if it's truly contradictory or just context-dependent
2. If contradictory: Store both with provenance, mark as inconsistent
3. Attempt resolution: Is one source more authoritative? Is there a missing context?
4. If unresolvable: Keep both, flag for human review or further investigation

**Self-consistency is the loss function:** Minimize contradictions while maximizing coverage of observed facts.

---

## Architecture

### Layer 1: Text Input with Provenance
Every piece of text that enters the system is tagged with:
- Source identifier
- Location (document, page, line, etc.)
- Context (fiction vs non-fiction, date written, author, etc.)
- Metadata (language, domain, etc.)

### Layer 2: LLM Parser
The LLM's role: **Linguistic → Conceptual mapping**

**Input:** Natural language sentence + provenance
**Output:** Graph operations (add/modify/link concepts) + parsed structure

**Example:**
```
Input: "The dog wasn't barking loudly at the mailman."
Source: [example.txt, line 5]

LLM parses to:
{
  "concepts": [
    {"id": "CONCEPT_DOG", "type": "instance"},
    {"id": "CONCEPT_BARKING", "type": "behavior"},
    {"id": "CONCEPT_MAILMAN", "type": "instance"}
  ],
  "properties": [
    {"concept": "CONCEPT_DOG", "property": "performing_behavior", "value": "CONCEPT_BARKING", "negated": true, "modifier": "loudly"},
    {"concept": "CONCEPT_BARKING", "property": "directed_at", "value": "CONCEPT_MAILMAN"}
  ],
  "provenance": {
    "source": "example.txt",
    "line": 5,
    "confidence": 0.92
  }
}
```

### Layer 3: Concept Graph Database
Graph structure storing:
- **Nodes:** Concepts (not words!)
- **Edges:** Relationships between concepts
- **Node properties:**
  - Possible values for each property dimension
  - Constraints on values
  - Provenance for each assertion
- **Edge types:**
  - is-a (taxonomic)
  - has-property
  - has-part
  - causes/enables/prevents
  - temporal (before/after)
  - contextual (occurs-in)
  - etc.

### Layer 4: Consistency Engine
Monitors the graph for:
- Logical contradictions
- Missing transitive relationships
- Violated constraints
- Redundant concepts (should these merge?)
- Over-split concepts (should these separate?)

**Operations:**
- **Merge concepts:** "Dog" and "chien" used identically across contexts → merge
- **Split concepts:** "Bank" used in incompatible ways → split into BANK_FINANCIAL and BANK_RIVER
- **Add implied relationships:** If A is-a B and B is-a C, ensure A is-a C exists
- **Flag contradictions:** Same property has mutually exclusive values from different sources

### Layer 5: Query & Reasoning Interface
Once the graph is built, you can:
- **Traverse:** "What are all behaviors in DOG-space?"
- **Query:** "Can a dog be a superhero?" → Check if that's in the observed DOG-space
- **Reason:** "If all dogs are mammals, and all mammals are warm-blooded, is this dog warm-blooded?" → Transitive inference
- **Explain:** "Why do you think dogs bark?" → Return all provenance supporting that claim
- **Update:** Add new text, watch the graph evolve

---

## Implementation Approach

### Phase 0: Proof of Concept
**Corpus:** Single short story or small text collection (children's books?)

**Goal:** Parse text into concept graph, verify it captures meaning

**Steps:**
1. Manually tag a few sentences with desired parse output (training examples)
2. Prompt LLM to parse new sentences in the same format
3. Build simple graph database (could start with JSON files or SQLite)
4. Visualize the resulting concept graph
5. Test: Can it answer questions about the text that require inference?

### Phase 1: Parser Development
**Goal:** Robust LLM-based parser that consistently outputs structured concept representations

**Challenges:**
- Prompt engineering for consistent output format
- Handling ambiguity (when should it create new concepts vs reference existing?)
- Confidence scoring (how certain is the parse?)
- Coreference resolution ("The dog... it..." → same instance)

**Approach:**
- Fine-tune or few-shot prompt an LLM specifically for this parsing task
- Structured output (JSON schema enforcement)
- Iterative refinement based on errors

### Phase 2: Concept Space Management
**Goal:** Automatically merge, split, and organize concepts based on usage patterns

**Challenges:**
- When do two linguistic tokens refer to the same concept?
- When should one concept split into two?
- How to represent concept hierarchies?

**Approach:**
- Clustering based on logical behavior (same relationships → same concept)
- Cross-lingual consistency (if "dog" and "chien" have identical relationships, merge)
- User feedback loop for ambiguous cases

### Phase 3: Consistency & Reasoning
**Goal:** Automatic consistency checking and inference

**Challenges:**
- Defining what constitutes a contradiction in natural language
- Handling context-dependent truth
- Computational complexity of consistency checking

**Approach:**
- Start with simple logical rules (transitivity, mutual exclusion)
- Probabilistic logic for uncertainty
- Mark contradictions rather than rejecting them - let human decide

### Phase 4: Scale & Integration
**Goal:** Handle large corpora, integrate with applications

**Challenges:**
- Graph size (millions of concepts)
- Query performance
- Incremental updates (new text shouldn't require rebuilding everything)
- Multi-user concurrent access

**Approach:**
- Real graph database (Neo4j, etc.)
- Indexing and caching strategies
- Streaming/incremental parsing
- API for external applications

---

## Why This Might Actually Work

### It Solves the Symbol Grounding Problem
Concepts aren't floating abstractions - they're grounded in actual text with provenance. You can always trace back to "what text made you think this?"

### It Separates Parsing from Knowledge
The LLM doesn't "know" things (hallucination risk). It only parses sentences. Knowledge emerges from the accumulated parsed structures across many texts.

### It's Language-Agnostic
The graph stores concepts, not words. Add a French text → same concepts get enriched, just with new linguistic labels.

### It's Inspectable & Editable
Unlike LLM weights, you can actually look at the graph, understand it, and correct it. "Oh, it merged these two concepts incorrectly - let me split them."

### It Handles Uncertainty Naturally
Multiple sources, contradictions, confidence scores - it's all explicit in the graph structure rather than muddled in parameters.

### It Can Reason
Once you have explicit structure, you can do actual logical inference in ways that pure LLMs can't reliably do.

---

## Why This Might Not Work

### LLMs Might Not Parse Consistently
Even with careful prompting, LLMs might parse the same sentence differently on different runs. This would make the graph unstable.

**Mitigation:** Use temperature=0, structured outputs, or fine-tuned models. Validate parses against each other.

### Concept Extraction Might Be Intractable
"What is a concept?" is philosophically fraught. Automatically discovering the right concept boundaries might be AI-complete.

**Mitigation:** Start with simple domains (children's stories), accept imperfection, allow human oversight.

### Consistency Checking Might Explode Computationally
Checking all pairs of facts for consistency in a million-node graph is O(n²) or worse.

**Mitigation:** Local consistency checks, probabilistic sampling, flag likely contradictions for review rather than exhaustive checking.

### It Might Just Reinvent Knowledge Graphs With Extra Steps
We already have knowledge graphs (DBpedia, Wikidata, etc.). Is this meaningfully different?

**Counter:** Yes - those are manually curated or extracted with brittle rules. This is automatic, provenance-grounded, and treats concepts as possibility spaces rather than single entities.

### AGI Might Require More Than Knowledge Representation
Even perfect knowledge graphs might not lead to general intelligence if intelligence requires embodiment, consciousness, social interaction, or something else entirely.

**Counter:** True. But even if it doesn't achieve AGI, a better knowledge representation system is still valuable.

---

## Success Criteria

### Minimal Success
Can parse a children's story, build a concept graph, and answer basic inference questions:
- "Is a dog an animal?" (requires is-a transitivity)
- "What did the dog do?" (requires event extraction)
- "Can dogs fly?" (requires checking DOG-space for flying behavior)

### Moderate Success
Can parse multiple texts across domains, maintain consistency, handle contradictions:
- Process both fiction and non-fiction
- Correctly distinguish CONCEPT_DRAGON_FICTIONAL from CONCEPT_KOMODO_DRAGON_REAL
- Flag when sources contradict
- Merge concepts across languages

### Strong Success
Demonstrates emergent reasoning capabilities:
- Can answer questions not explicitly in the text by combining facts
- Can explain its reasoning with provenance chain
- Can identify gaps in its knowledge ("I don't know if dogs can swim - none of my sources mention it")
- Can update beliefs when new information arrives

### Transformative Success
Becomes genuinely useful for knowledge work:
- Researchers use it to organize literature reviews
- Writers use it to maintain consistency in fictional worlds
- Students use it to build understanding of domains
- Outperforms pure LLMs on reasoning benchmarks

---

## First Experiment: Simple Implementation

**Corpus:** One Sherlock Holmes short story ("The Adventure of Silver Blaze" - the "dog that didn't bark")

**Tools:**
- LLM API (Claude/GPT-4) for parsing
- Python for orchestration
- NetworkX or simple graph library
- SQLite for provenance tracking

**Process:**
1. Split story into sentences
2. For each sentence, prompt LLM: "Parse this into concepts and relationships (JSON format)"
3. Build graph incrementally
4. Visualize the resulting graph
5. Test queries:
   - "What happened in the story?" (event sequence)
   - "Why was the dog significant?" (requires understanding the negation/silence)
   - "What are the properties of the horse?" (aggregate from multiple mentions)

**Expected outcome:** Either a coherent graph that captures story meaning, or immediate failure modes that reveal what needs fixing.

**Time estimate:** Could have a basic prototype in a weekend. Rough, but functional enough to test the core idea.

---

## Open Questions

1. **How do you handle temporal information?** Events happen in sequence, states change over time. How does this fit in the graph?

2. **What about nested/hypothetical contexts?** "If the dog had barked, the thief would have fled" - how do you represent counterfactuals?

3. **How do you handle vague/gradual properties?** "The dog was fairly large" - is this a discrete value or continuous?

4. **Can you extract causation reliably?** "The dog barked, and the mailman left" - correlation or causation?

5. **How do you handle metaphor/figurative language?** "He's a dog" (meaning: mean person) vs "He has a dog" (literal)

6. **What's the UI for exploring the graph?** How do humans interact with this to understand/correct/query it?

---

## Conclusion

This approach combines:
- **LLMs** for linguistic understanding (parsing)
- **Provenance** for grounding and traceability
- **Concept spaces** for rich representation
- **Self-consistency** as organizing principle

It's ambitious but feels more tractable than either:
- Pure symbolic AI (can't handle natural language)
- Pure neural (black box, hallucinates)

The key innovation: **Use LLMs as parsers, not knowledge sources.** They map language to concepts, the graph stores the knowledge.

Worth building? Absolutely. Will it lead to AGI? Unclear. But it might lead to something genuinely more capable than current knowledge graphs or pure LLMs for certain reasoning tasks.

And the failure modes will be educational.
