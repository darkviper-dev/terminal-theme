import pdfplumber
import csv
import re

pdf_path = "example.pdf"              # Change this to your PDF file
output_csv_path = "requirements.csv"  # Output CSV file

# --- Helper Functions ---

def clean_block(block_text, table_texts):
    cleaned_text = block_text

    # Remove any table content from block text
    table_texts_sorted = sorted(set(table_texts), key=len, reverse=True)
    for t_text in table_texts_sorted:
        pattern = re.escape(t_text.strip())
        cleaned_text = re.sub(pattern, '', cleaned_text, flags=re.MULTILINE)

    # Remove URLs and page numbers
    cleaned_text = re.sub(r'https?://\S+', '', cleaned_text)
    cleaned_text = re.sub(r'Page \d+ of \d+', '', cleaned_text, flags=re.IGNORECASE)

    # Normalize spacing
    cleaned_text = re.sub(r'\n\s*\n', '\n', cleaned_text)
    cleaned_text = cleaned_text.strip()
    return cleaned_text

def extract_req_id(line):
    match = re.match(r'(REQ_[^\s:]+)', line)
    return match.group(1) if match else None

def is_metadata_line(line):
    line = line.strip()
    if not line or line.startswith("REQ_"):
        return False
    if ':' in line:
        key = line.split(':', 1)[0]
        return ' ' not in key and len(key) <= 20
    return False

def is_heading_or_table_header(line):
    line = line.strip()
    if not line:
        return False
    if line.isupper():
        return True
    if re.match(r'^\d+(\.\d+)?[\s\-]+[A-Z ]+$', line):
        return True
    if len(line.split()) <= 5 and line == line.title():
        return True
    return False

def extract_req_id_and_description(block):
    match = re.match(r'(REQ_[^\s:]+)[\s:]*([\s\S]*)', block.strip())
    if match:
        return match.group(1).strip(), match.group(2).strip()
    return None, block.strip()

# --- Process PDF and Extract REQs ---

csv_rows = []

with pdfplumber.open(pdf_path) as pdf:
    for i, page in enumerate(pdf.pages):
        all_tables = page.extract_tables()
        table_texts = []
        table_req_ids = set()

        # Extract REQ-related tables
        for table in all_tables:
            flat_cells = [cell for row in table for cell in row if cell]
            if any("REQ_" in cell for cell in flat_cells):
                table_texts.extend(flat_cells)
                for cell in flat_cells:
                    rid = extract_req_id(cell)
                    if rid:
                        table_req_ids.add(rid)

        # Extract REQ blocks from text
        text = page.extract_text()
        lines = text.split('\n') if text else []

        index = 0
        while index < len(lines):
            line = lines[index].strip()
            if line.startswith("REQ_"):
                block_lines = [line]
                index += 1
                while index < len(lines):
                    next_line = lines[index].strip()
                    if (
                        next_line.startswith("REQ_") or
                        not next_line or
                        is_metadata_line(next_line) or
                        is_heading_or_table_header(next_line)
                    ):
                        break
                    block_lines.append(next_line)
                    index += 1

                block_text = "\n".join(block_lines)
                cleaned_block = clean_block(block_text, table_texts)

                first_line = cleaned_block.split('\n')[0] if cleaned_block else ""
                req_id = extract_req_id(first_line)

                if cleaned_block and "REQ_" in cleaned_block and req_id not in table_req_ids:
                    req_id, description = extract_req_id_and_description(cleaned_block)
                    if req_id and description:
                        csv_rows.append({
                            "Request ID": req_id,
                            "Description": description
                        })
            else:
                index += 1

# --- Save CSV ---

with open(output_csv_path, 'w', newline='', encoding='utf-8') as csvfile:
    fieldnames = ["Request ID", "Description"]
    writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(csv_rows)

print(f"✅ CSV created at '{output_csv_path}' with {len(csv_rows)} entries.")
