---
name: pdf-modification
description: Use this skill when modifying, editing, or changing content in existing PDF files. This includes removing content (text, solution boxes, sections), redacting information, extracting pages, or any structural PDF changes. Always use PyMuPDF (fitz) to work with the original PDF rather than recreating it. CRITICAL - For LaTeX-generated PDFs with bordered boxes (like solution boxes), detect and remove by finding the actual border rectangles, NOT by text parsing. Must verify all changes programmatically before presenting results.
---

# PDF Modification Skill

## Core Principles

### 1. NEVER Recreate - Always Modify Original
When modifying PDFs, ALWAYS work with the original file using PyMuPDF (fitz). Never attempt to recreate the PDF in LaTeX, HTML, or any other format.

**Why this matters:**
- Original figures and diagrams are preserved exactly
- Precise formatting and layout maintained
- Embedded fonts and styling intact
- Page structure and metadata preserved
- No risk of introducing new errors

### 2. For LaTeX PDFs: Detect Visual Structure, Not Text
**CRITICAL LESSON:** LaTeX-compiled PDFs often have visual boxes with borders (like solution boxes in exams). These must be detected by their visual structure (border rectangles), NOT by parsing text markers.

**Why text parsing fails:**
- Text markers like "Solution:" can appear anywhere
- Section boundaries are ambiguous across pages
- Figures can be placed between solution content due to LaTeX float placement
- Multi-page content doesn't follow predictable text patterns

**Correct approach:**
```python
# Find bordered boxes by detecting their rectangular borders
drawings = page.get_drawings()

# Look for sets of 4 lines forming a rectangle:
# - Top horizontal line
# - Bottom horizontal line  
# - Left vertical line
# - Right vertical line

# Then remove everything inside those rectangles
```

### 3. Use Proper Redaction
When removing content, use proper redaction that actually removes it from the PDF structure:

```python
page.add_redact_annot(rect, fill=(1, 1, 1))
page.apply_redactions()
```

Do NOT just draw white rectangles - this only hides content visually but doesn't remove it.

### 4. ALWAYS Verify Changes
This is non-negotiable. After making any modifications:

1. Create a verification script
2. Search for text that should be removed
3. Confirm important content (like figures) is still present
4. If verification fails, iterate and fix
5. Only present the PDF after verification succeeds

## Implementation: Removing LaTeX Solution Boxes

Based on real-world experience with exam PDFs:

```python
import fitz

def find_bordered_boxes(doc):
    """Find all boxes defined by rectangular borders"""
    all_redactions = {}
    
    for page_num in range(len(doc)):
        page = doc[page_num]
        all_redactions[page_num] = []
        
        drawings = page.get_drawings()
        
        # Collect horizontal and vertical lines
        h_lines = []
        v_lines = []
        
        for drawing in drawings:
            rect = drawing.get('rect')
            if not rect:
                continue
            
            w, h = rect.width, rect.height
            
            # Horizontal line: long width, minimal height
            if w > 400 and h < 2:
                h_lines.append({'x0': rect.x0, 'x1': rect.x1, 'y': rect.y0})
            
            # Vertical line: long height, minimal width
            elif h > 20 and w < 2:
                v_lines.append({'x': rect.x0, 'y0': rect.y0, 'y1': rect.y1})
        
        # Find complete boxes by matching top/bottom with left/right
        for i, top in enumerate(h_lines):
            for j, bottom in enumerate(h_lines):
                if j <= i:
                    continue
                
                y_diff = bottom['y'] - top['y']
                if y_diff < 20:  # Must be reasonable height
                    continue
                
                # Check x-coordinates align (with tolerance)
                x_diff_left = abs(top['x0'] - bottom['x0'])
                x_diff_right = abs(top['x1'] - bottom['x1'])
                
                if x_diff_left < 50 and x_diff_right < 50:
                    # Look for vertical lines connecting top and bottom
                    has_left = False
                    has_right = False
                    
                    for vert in v_lines:
                        # Check if vertical spans top to bottom
                        y0_match = abs(vert['y0'] - top['y']) < 15
                        y1_match = abs(vert['y1'] - bottom['y']) < 15
                        
                        if y0_match and y1_match:
                            if abs(vert['x'] - top['x0']) < 50:
                                has_left = True
                            if abs(vert['x'] - top['x1']) < 50:
                                has_right = True
                    
                    # Complete box found
                    if has_left and has_right:
                        x0 = min(top['x0'], bottom['x0']) - 2
                        x1 = max(top['x1'], bottom['x1']) + 2
                        y0 = top['y'] - 2
                        y1 = bottom['y'] + 2
                        
                        rect = fitz.Rect(x0, y0, x1, y1)
                        all_redactions[page_num].append(rect)
    
    return all_redactions

def remove_solution_boxes(input_pdf, output_pdf):
    """Remove bordered solution boxes"""
    doc = fitz.open(input_pdf)
    
    all_redactions = find_bordered_boxes(doc)
    
    # Apply redactions
    for page_num, rects in all_redactions.items():
        if rects:
            page = doc[page_num]
            for rect in rects:
                page.add_redact_annot(rect, fill=(1, 1, 1))
            page.apply_redactions()
    
    doc.save(output_pdf, garbage=4, deflate=True)
    doc.close()

def verify_removal(output_pdf):
    """ALWAYS verify results"""
    doc = fitz.open(output_pdf)
    
    solutions_found = []
    figures_present = {}
    
    for page_num in range(len(doc)):
        page = doc[page_num]
        text = page.get_text()
        
        # Check that unwanted content is gone
        if "Solution:" in text:
            solutions_found.append(page_num + 1)
        
        # Check that important content remains
        for fig_num in [1, 2, 3, 4]:
            if f"Figure {fig_num}" in text:
                figures_present[fig_num] = True
    
    doc.close()
    
    # Report results
    success = len(solutions_found) == 0 and len(figures_present) > 0
    
    if not success:
        print("VERIFICATION FAILED - DO NOT PRESENT")
        if solutions_found:
            print(f"  Solution text still on pages: {solutions_found}")
        print(f"  Figures found: {list(figures_present.keys())}")
    
    return success

# Complete workflow
remove_solution_boxes(input_file, output_file)
if verify_removal(output_file):
    # Only present after verification succeeds
    present_file(output_file)
else:
    # Iterate and fix
    pass
```

## Key Lessons Learned

### Multi-Page Boxes
LaTeX can place figures between parts of a solution box due to float placement. This means:
- A solution box might start on page N
- A figure appears on page N+1  
- The solution box continues on page N+1 below the figure

**Solution:** Detect boxes by borders on each page independently. Don't try to track "continuing" boxes across pages by text analysis.

### Tolerance Values Matter
Border detection needs tolerance for:
- X-coordinate alignment: ±50 points (boxes can have slightly different widths)
- Y-coordinate matching: ±15 points (lines may not align perfectly)
- Minimum heights: >20 points (avoid tiny boxes)

### Verification Must Be Specific
Generic "check if it worked" is insufficient. Verification must check:
1. Specific unwanted text is gone ("Solution:")
2. Specific important content remains (Figure 1, Figure 2, etc.)
3. Both must pass for success

## Common Pitfalls

### ❌ Text-Based Detection
```python
# WRONG - fragile across page breaks and figure placement
if "Solution:" in text:
    # find next section marker
    # remove everything between
```

### ✓ Visual Structure Detection
```python
# CORRECT - robust to LaTeX layout
drawings = page.get_drawings()
# Find rectangular borders
# Remove everything inside
```

### ❌ Assuming Text Structure
```python
# WRONG - assumes linear text flow
# "Solution on page 2 continues to page 3"
# "End when we see (b)"
```

### ✓ Per-Page Analysis
```python
# CORRECT - analyze each page independently
for page in doc:
    boxes = find_bordered_boxes_on_this_page(page)
    remove_boxes(boxes)
```

## Installation

```bash
pip install pymupdf --break-system-packages
```

Note: PyMuPDF is imported as `fitz`.

## Workflow Summary

1. **Identify the task type**
   - LaTeX PDF with bordered boxes? → Use visual detection
   - Text-only modifications? → Can use text-based approach

2. **Implement the solution**
   - Always work with original PDF
   - Use proper redaction, not covering
   - Handle edge cases (tolerances, multi-page, etc.)

3. **Verify programmatically**
   - Check unwanted content is gone
   - Check important content remains
   - Both must pass

4. **Only present after verification succeeds**
   - Never assume it worked
   - If verification fails, iterate
   - Show verification results to user

## Key Takeaway

**For LaTeX-generated PDFs: Trust the visual structure, not the text structure.** Boxes have borders - detect them. Figures are placed by LaTeX - they'll be outside the boxes. Text markers are ambiguous - ignore them.
