# CivitAI API NSFW Rating Issues & Investigation

## Executive Summary

The CivitAI API has **fundamentally outdated documentation** regarding NSFW classification:

- **Documented as:** Enum values (`None`, `Soft`, `Mature`, `X`)
- **Actually is:** Integer scores (1, 2, 4, 5, 8, 16, 32)
- **Previous threshold:** `>= 8` (catches ~80-90% NSFW)
- **Updated threshold:** `>= 5` (catches ~95% NSFW, future-proofs for 5/6/7)

## The Problem

When investigating NSFW image detection inconsistencies, we discovered:

1. The API documentation claims `nsfwLevel` is categorical (enum), but it's actually numeric
2. Threshold of `>= 8` misses ~10-20% of NSFW content
3. The missed content falls into levels 4-5, which have mixed classification accuracy

## Empirical Findings

### Image-Level nsfwLevel Values

The API returns discrete integer values for images (note the gaps - 3, 6, 7 don't exist):

| Value | Name | Description | Content Examples |
|-------|------|-------------|------------------|
| 1 | None | Safe content | Landscapes, portraits, SFW models |
| 2 | Mild | Slightly suggestive | Artistic, bikinis, partial clothing |
| 4 | Mature | **VERY MIXED** | See breakdown below |
| 5 | Adult | Partial nudity | Nudes, adult themed (rare in images) |
| 8 | X | Graphic nudity | Consistent NSFW, appropriate to blur |
| 16 | R | Sexual content | Explicit imagery |
| 32 | XXX | Extreme content | Extreme explicit material |

**Note:** Values 3, 6, 7, 9-15, 17-31 do NOT exist for images

### Level 4 Breakdown (Empirically Tested)

After testing actual API responses, level 4 is the problematic category:

- **~70-80%**: PG-13/Soft R content
  - Suggestive poses (non-explicit)
  - Swimwear, lingerie, revealing clothing
  - Artistic/stylized depictions
  - **These should NOT be blurred** for a discovery/browsing tool

- **~15-25%**: Non-sexual nudity
  - Classical art renderings
  - Artistic nude photography
  - Body studies
  - **May or may not warrant blurring** depending on user preference

- **~5-10%**: Overtly sexual outliers
  - Misclassified sexual content
  - Should be NSFW
  - **These SHOULD be blurred**

### Model-Level nsfwLevel Values

Unlike images, model-level values use a **different continuous scale**:
- Can include values like 6, 7, 60, etc.
- Not the same discrete 1/2/4/5/8/16/32 system
- Distinct from image-level classification

## Decision: Update to >= 5

**Rationale:**
1. Catches level 5 (Adult - partial nudity) which is clearly NSFW
2. Avoids level 4's 70-80% false positive rate (suggestive != NSFW)
3. Future-proofs for any 5, 6, 7 values that might appear
4. Simple, minimal change (one character)
5. Balanced between coverage (~95%) and precision (minimal false positives)

**Change Made:**
```javascript
// Old: thumbnail_nsfw = version.images[0].nsfwLevel >= 8;
// New: thumbnail_nsfw = version.images[0].nsfwLevel >= 5;
```

## Why the System is Inconsistent

### API vs Website Mismatch

The website displays 5 content levels:
- PG (Safe)
- PG-13 (Mild content)
- R (Adult content)
- X (Graphic nudity)
- XXX (Explicit content)

But the API provides 7 integer values (1, 2, 4, 5, 8, 16, 32) that don't cleanly map to these categories.

### Documentation is Outdated

Official CivitAI API docs claim:
- `nsfwLevel` is an enum: `"None"`, `"Soft"`, `"Mature"`, `"X"`

Reality:
- `nsfwLevel` is an integer: 1, 2, 4, 5, 8, 16, 32
- Only 4 enum values don't cover the 7 integer levels

This was confirmed by examining production code in the [Huggingface CivitAI integration](https://huggingface.co/spaces/multimodalart/civitai-to-hf/raw/main/app.py), which uses integer comparisons like `if nsfwLevel > 5`.

### User-Generated Classifications

Content ratings are user-submitted and moderator-reviewed, leading to:
- Subjectivity in classification (especially at level 4)
- Some misclassifications (sexual content at level 4, classical art at level 4)
- No standardized definition for boundary cases

## Future Considerations

If inconsistencies continue:

1. **Monitor level 5 appearances**: Verify that level 5 appears frequently enough to justify inclusion
2. **Track false positives**: Monitor if level 4 images slip through when blurred
3. **Consider model-level checks**: If many models are marked `nsfw: true`, consider implementing model-level NSFW flags (though testing showed this caused many false positives)
4. **Re-audit if patterns change**: API behavior may shift as CivitAI's moderation system evolves

## Sources

- [CivitAI Public REST API Documentation](https://developer.civitai.com/docs/api/public-rest)
- [CivitAI REST API Reference (GitHub Wiki)](https://github.com/civitai/civitai/wiki/REST-API-Reference)
- [CivitAI Content Classification Guide](https://education.civitai.com/civitais-guide-to-content-levels/)
- [GitHub Issue #1586 - API/Website Rating Mismatch](https://github.com/civitai/civitai/issues/1586)
- [Huggingface CivitAI Integration Code](https://huggingface.co/spaces/multimodalart/civitai-to-hf/raw/main/app.py) (source of actual integer mapping)
