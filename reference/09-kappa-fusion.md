# Kappa Graph Fusion

How the OSINT operational graph and the Kappa semantic graph reinforce each other.

## Two Graphs, Different Jobs

| | OSINT Graph (this platform) | Kappa Graph |
|---|---|---|
| **Domain** | Operational — what's happening | Semantic — what things mean |
| **Vertices** | Entities, Events, Observations, Locations | Concepts |
| **Edges** | NEAR, OBSERVED_AT, RELATED_TO | IMPLIES, SUPPORTS, CONTRADICTS |
| **Edge weight** | Confidence (observation quality) | Confidence (epistemic grounding) |
| **Temporal** | Epochs, timestamps, validity windows | Document provenance, first_seen/last_updated |
| **Scale** | ~140K vertices steady state, high churn | Growing corpus, low churn |
| **Query style** | "What's near this carrier right now?" | "What does 'supply chain disruption' relate to?" |

Neither graph can answer the other's questions. Together, they close the loop between observation and meaning.

## The Bridge: Foreign References

Every OSINT event that gets promoted into Kappa carries a foreign reference back to its source:

```cypher
-- In Kappa's graph
CREATE (c:Concept {
  concept_id: 'supply-chain-disruption-hormuz-2026q1',
  name: 'Strait of Hormuz supply chain disruption (2026 Q1)',
  grounding_score: 0.72,
  external_refs: [
    'osint://events/gdelt-78901',
    'osint://events/gdelt-78902',
    'osint://events/rss:sha256:abc123',
    'osint://epoch-window/hormuz-march-2026'
  ]
})
```

And OSINT entities that have been referenced by Kappa concepts carry a back-reference:

```cypher
-- In OSINT graph
MATCH (ev:Event {event_id: 'gdelt-78901'})
SET ev.kappa_refs = ['kappa://concepts/supply-chain-disruption-hormuz-2026q1']
```

Bidirectional. Either graph can resolve the other's references.

## Semantic Grounding from Observable Events

Kappa concepts have grounding scores — a measure of how well-supported a claim is. Currently, grounding comes from documents (papers, articles, web sources). With the OSINT bridge, grounding also comes from **observable real-world events**.

### How it works

1. **Correlation scan** detects a cluster of OSINT events (e.g., tanker diversions + sanctions news + military activity near Hormuz)

2. **The cluster gets promoted** to Kappa as a concept or as supporting evidence for an existing concept:

```python
async def promote_to_kappa(situation: EmergingSituation):
    """Push an OSINT situation into Kappa's semantic graph."""

    # Find or create the concept in Kappa
    concept = await kappa.find_or_create_concept(
        name=situation.summary,
        properties={
            'source': 'osint_promotion',
            'osint_event_count': len(situation.events),
            'osint_max_criticality': situation.max_score,
            'osint_epoch_window': situation.epoch_range,
        }
    )

    # Create SUPPORTS edges from the OSINT events to the concept
    for event in situation.events:
        await kappa.create_edge(
            from_concept=f'osint-event:{event.event_id}',
            to_concept=concept.concept_id,
            edge_type='SUPPORTS',
            properties={
                'confidence': event_to_confidence(event),
                'source': 'osint_observation',
                'external_ref': f'osint://events/{event.event_id}',
                'observation_timestamp': event.timestamp,
            }
        )

    # Kappa recalculates grounding score
    # Now this concept is grounded not just by documents but by observable events
    await kappa.recalculate_grounding(concept.concept_id)
```

3. **Kappa's grounding score updates.** A concept that was at 0.55 grounding from documents alone might jump to 0.82 when backed by 7 correlated OSINT observations.

### The Grounding Equation

Kappa's existing grounding formula:
```
grounding = (supporting_count - contradicting_count) / total_relationships
```

Extended with OSINT weight:
```
grounding = (doc_supports + osint_supports × osint_weight - contradictions) / total

where osint_weight reflects:
  - recency (recent observations weigh more)
  - observation_confidence (direct ADS-B > news scraping > GDELT geocoding)
  - corroboration (multiple independent sources > single source)
```

OSINT observations are empirical evidence. A tanker actually being at Hormuz is stronger backing than an article saying tankers are at Hormuz. The weight reflects that.

## Reverse Traversal: Kappa → OSINT

Claude asks Kappa: "What concepts relate to 'naval power projection'?"

Kappa returns a semantic neighborhood:
```
naval_power_projection
  ├── IMPLIES → carrier_strike_group_deployment (0.91)
  ├── IMPLIES → freedom_of_navigation (0.78)
  ├── SUPPORTS → deterrence_theory (0.65)
  └── RELATED → supply_chain_security (0.58)
```

Some of these concepts have `external_refs` pointing to OSINT data. Claude can now resolve them:

```python
@mcp_tool("resolve_kappa_concept")
async def resolve_kappa_concept(concept_id: str) -> str:
    """Look up a Kappa concept and find its OSINT backing events."""

    concept = await kappa.get_concept(concept_id)
    osint_refs = [ref for ref in concept.external_refs if ref.startswith('osint://')]

    if not osint_refs:
        return f"Concept '{concept.name}' has no OSINT backing events."

    lines = [f"## OSINT Backing for: {concept.name}",
             f"**Kappa grounding:** {concept.grounding_score:.2f}",
             f"**OSINT references:** {len(osint_refs)}", ""]

    for ref in osint_refs:
        if ref.startswith('osint://events/'):
            event_id = ref.split('/')[-1]
            event = await osint_graph.get_event(event_id)
            if event:
                lines.append(f"- **{event.event_type}** at ({event.lat:.2f}, {event.lng:.2f}) "
                            f"on {event.timestamp.strftime('%Y-%m-%d %H:%M')} UTC")
                lines.append(f"  {event.description}")

        elif ref.startswith('osint://epoch-window/'):
            window_name = ref.split('/')[-1]
            window = await osint_graph.get_window(window_name)
            if window:
                lines.append(f"- **Epoch window** '{window_name}': "
                            f"{window.entity_count} entities, {window.event_count} events")

    return "\n".join(lines)
```

Claude gets:
```
## OSINT Backing for: Strait of Hormuz supply chain disruption (2026 Q1)
**Kappa grounding:** 0.82
**OSINT references:** 5

- **gdelt_event** at (26.55, 56.25) on 2026-03-10 14:30 UTC
  Iran announces new naval exercises in Strait of Hormuz
- **gdelt_event** at (26.42, 56.38) on 2026-03-11 09:15 UTC
  Oil prices spike on Hormuz transit concerns
- **news_article** at (26.60, 56.20) on 2026-03-12 16:00 UTC
  Reuters: Three tankers diverted from Hormuz route
- **Epoch window** 'hormuz-march-2026': 34 entities, 12 events
```

Now Claude can narrate: "The concept 'supply chain disruption at Hormuz' is grounded at 0.82 in Kappa, backed by 5 observable OSINT events including tanker diversions and naval exercise announcements. The trajectory data shows 12 cargo vessels altered course between March 10-14..."

## Semantic Discovery of Unknown Connections

This is where it gets interesting. Kappa's semantic graph can find connections that the OSINT graph can't — because OSINT only knows about co-location and temporal proximity, but Kappa knows about *meaning*.

### Example: Unrelated Events Connected by Concept

Two OSINT events that look unrelated:
1. A Russian cargo vessel docks at Tartus, Syria (March 10)
2. An unusual volume of satellite phone traffic detected in eastern Libya (March 12)

OSINT graph: these are 1,500 km apart, different entity types, no temporal overlap. No NEAR edge. No RELATED_TO edge. The correlation scan misses it.

But Kappa has:
```
russian_military_logistics
  ├── IMPLIES → tartus_naval_facility (0.89)
  ├── IMPLIES → wagner_group_resupply (0.71)
  └── SUPPORTS → libyan_civil_war_intervention (0.65)

wagner_group_resupply
  ├── IMPLIES → encrypted_communications_spike (0.73)
  └── RELATED → libyan_eastern_front (0.68)
```

Claude traverses Kappa: "Russian cargo at Tartus" → `russian_military_logistics` → `wagner_group_resupply` → `encrypted_communications_spike` → matches the Libya SIGINT event.

Kappa provides the semantic bridge. OSINT provides the timestamps and coordinates. Together: "Based on historical patterns, Russian cargo arrivals at Tartus have preceded Wagner Group resupply operations (Kappa grounding: 0.71), which correlate with communications spikes in Eastern Libya (0.73). The current Tartus docking on March 10 and the Libya SIGINT detection on March 12 fit this pattern. Recommend monitoring [specific coordinates] for logistics activity in the next 48-72 hours."

Neither graph alone produces that conclusion.

## Kappa Properties Enhancement

For this fusion to work, Kappa needs one addition to its concept model — **external reference properties**:

```python
# Addition to Kappa's Concept vertex schema
{
    "concept_id": "supply-chain-disruption-hormuz-2026q1",
    "name": "...",
    "grounding_score": 0.82,
    # NEW: external references to other systems
    "external_refs": [
        "osint://events/gdelt-78901",
        "osint://epoch-window/hormuz-march-2026"
    ],
    # NEW: reference metadata
    "external_ref_metadata": {
        "osint://events/gdelt-78901": {
            "system": "shadowbroker",
            "type": "event",
            "added_at": "2026-03-14T23:00:00Z",
            "added_by": "auto_promotion"
        }
    }
}
```

And a new edge type for cross-system grounding:

```
GROUNDED_BY  — Concept → ExternalRef
  properties:
    system: 'shadowbroker'
    ref_uri: 'osint://events/gdelt-78901'
    confidence: 0.85
    observation_type: 'direct' | 'inferred' | 'correlated'
    epoch: 47250
    timestamp: '2026-03-14T23:00:00Z'
```

This is a lightweight addition — Kappa's graph_accel already handles arbitrary edge types, and the accelerator loads all edge labels by default. No schema migration needed on the AGE side, just new label conventions.

## Federation Sync

The sync between platforms is event-driven, not bulk:

```python
class KappaSync:
    """Pushes significant OSINT findings to Kappa."""

    async def on_emerging_situation(self, situation: EmergingSituation):
        """Promote emerging situations to Kappa concepts."""
        if situation.max_score >= self.promotion_threshold:
            await self.promote_to_kappa(situation)

    async def on_analysis_complete(self, report: Report):
        """Push analysis reports to Kappa as supporting evidence."""
        # Find relevant Kappa concepts via embedding similarity
        related = await kappa.semantic_search(report.summary, top_k=5)
        for concept in related:
            if concept.similarity > 0.7:
                await kappa.create_edge(
                    from_concept=f'osint-report:{report.id}',
                    to_concept=concept.concept_id,
                    edge_type='SUPPORTS',
                    properties={
                        'confidence': concept.similarity,
                        'source': 'osint_analysis',
                        'external_ref': f'osint://reports/{report.id}',
                    }
                )
                await kappa.recalculate_grounding(concept.concept_id)

    async def on_score_change(self, event_id: str, old_score: float, new_score: float):
        """Update Kappa grounding when OSINT scores change."""
        if abs(new_score - old_score) < 0.1:
            return  # not material

        kappa_refs = await osint_graph.get_kappa_refs(event_id)
        for ref in kappa_refs:
            await kappa.update_edge_confidence(
                ref_uri=f'osint://events/{event_id}',
                new_confidence=osint_score_to_confidence(new_score)
            )
            await kappa.recalculate_grounding(ref.concept_id)
```

Not a firehose. Only promotions above threshold, analysis reports with high semantic match, and material score changes. Kappa's graph stays clean.

## The Intelligence Cycle

```
      ┌─────────────────────────────────────────────────────┐
      │                                                       │
      ▼                                                       │
 ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐ │
 │ Collect  │───▶│ Correlate │───▶│ Promote │───▶│  Ground  │ │
 │ (OSINT)  │    │ (OSINT    │    │ (→Kappa) │    │ (Kappa   │ │
 │          │    │  graph)   │    │          │    │  scores) │ │
 └─────────┘    └──────────┘    └─────────┘    └────┬─────┘ │
                                                     │       │
                                                     ▼       │
                                                ┌──────────┐ │
                                                │ Traverse  │ │
                                                │ (Kappa    │ │
                                                │ semantic) │ │
                                                └────┬─────┘ │
                                                     │       │
                                                     ▼       │
                                                ┌──────────┐ │
                                                │ Discover  │ │
                                                │ (cross-   │ │
                                                │ reference)│ │
                                                └────┬─────┘ │
                                                     │       │
                                                     ▼       │
                                                ┌──────────┐ │
                                                │ Direct   │──┘
                                                │ (new     │
                                                │ collection│
                                                │ targets) │
                                                └──────────┘
```

1. **Collect** — OSINT collectors gather raw data
2. **Correlate** — OSINT graph finds spatial/temporal patterns
3. **Promote** — significant findings push to Kappa as concepts/evidence
4. **Ground** — Kappa's grounding scores update with empirical backing
5. **Traverse** — Claude explores Kappa's semantic graph, finds conceptual connections
6. **Discover** — foreign references resolve back to OSINT data for evidence
7. **Direct** — discoveries suggest new collection targets → back to step 1

The cycle tightens over time. As more OSINT backs more Kappa concepts, the semantic graph becomes more grounded. As Kappa identifies more patterns, it directs more focused collection. This is how institutional intelligence works, except the institution is a graph.
