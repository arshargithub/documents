# Normalized Document Artifact Samples (Pre-Segmentation)

Below are detailed sample JSON structures / canonical artifact shapes for the **pre-segmentation normalized representation** for **XLSX, DOCX, PDF, and EML**.

These are not meant to be rigid standards you must copy exactly. They are practical canonical contracts for the parser layer output so that downstream segmentation can be format-agnostic where possible.

Design goals:
- preserve source structure
- preserve locators/provenance
- keep enough metadata for segmentation/chunking
- avoid baking in extraction-schema knowledge
- keep raw parser weirdness behind a stable internal contract

## 1. Common envelope for all normalized document artifacts

Every normalized document artifact should share a common top-level envelope.

```json
{
  "artifact_version": "1.0",
  "normalization_version": "1.0",
  "request_id": "req_12345",
  "document_id": "doc_abc123",
  "source": {
    "s3_uri": "s3://bucket/requests/req_12345/raw/file.pdf",
    "original_filename": "claim_form.pdf",
    "mime_type": "application/pdf",
    "extension": "pdf",
    "checksum_sha256": "..."
  },
  "document_type": "pdf",
  "parser": {
    "parser_name": "pdf_parser_v1",
    "parser_version": "1.3.0",
    "parsed_at": "2026-03-20T14:12:00Z"
  },
  "document_stats": {
    "size_bytes": 1842291,
    "page_count": 3,
    "sheet_count": null,
    "block_count": 42,
    "table_count": 2,
    "text_char_count": 18342,
    "ocr_applied": false
  },
  "content": {},
  "quality": {
    "status": "ok",
    "warnings": [],
    "errors": []
  }
}
```

Fields will differ by format, but this envelope should stay stable.

## 2. PDF normalized JSON schema (pre-segmentation)

### Design notes
For PDF, the parser should preserve:
- page boundaries
- block order
- block types
- tables
- coordinates if available
- detected repeated headers/footers if possible

### Sample shape

```json
{
  "artifact_version": "1.0",
  "normalization_version": "1.0",
  "request_id": "req_12345",
  "document_id": "doc_pdf_001",
  "source": {
    "s3_uri": "s3://bucket/requests/req_12345/raw/claim_form.pdf",
    "original_filename": "claim_form.pdf",
    "mime_type": "application/pdf",
    "extension": "pdf",
    "checksum_sha256": "abc123"
  },
  "document_type": "pdf",
  "parser": {
    "parser_name": "pdf_parser_v1",
    "parser_version": "1.3.0",
    "parsed_at": "2026-03-20T14:12:00Z"
  },
  "document_stats": {
    "size_bytes": 1842291,
    "page_count": 3,
    "block_count": 42,
    "table_count": 2,
    "text_char_count": 18342,
    "ocr_applied": false
  },
  "content": {
    "pages": [
      {
        "page_number": 1,
        "dimensions": {
          "width": 612,
          "height": 792,
          "unit": "pt"
        },
        "reading_order": [
          "blk_1",
          "blk_2",
          "blk_3",
          "tbl_1",
          "blk_4"
        ],
        "blocks": [
          {
            "block_id": "blk_1",
            "block_type": "header",
            "text": "Claim Submission Form",
            "bbox": [72, 45, 280, 68],
            "style": {
              "font_family": "Helvetica-Bold",
              "font_size": 16,
              "bold": true,
              "italic": false
            },
            "page_position_hint": "top",
            "parser_confidence": 0.98
          },
          {
            "block_id": "blk_2",
            "block_type": "paragraph",
            "text": "Customer Name: John Smith",
            "bbox": [72, 110, 290, 128],
            "style": {
              "font_family": "Helvetica",
              "font_size": 11,
              "bold": false,
              "italic": false
            },
            "page_position_hint": "body",
            "parser_confidence": 0.99
          },
          {
            "block_id": "blk_3",
            "block_type": "paragraph",
            "text": "Policy Number: PL-88291",
            "bbox": [72, 130, 270, 148],
            "style": {
              "font_family": "Helvetica",
              "font_size": 11,
              "bold": false,
              "italic": false
            },
            "page_position_hint": "body",
            "parser_confidence": 0.99
          },
          {
            "block_id": "blk_4",
            "block_type": "footer",
            "text": "Page 1 of 3",
            "bbox": [270, 760, 340, 775],
            "style": {
              "font_family": "Helvetica",
              "font_size": 9,
              "bold": false,
              "italic": false
            },
            "page_position_hint": "bottom",
            "parser_confidence": 0.97
          }
        ],
        "tables": [
          {
            "table_id": "tbl_1",
            "bbox": [70, 180, 540, 360],
            "title": "Claim Details",
            "header_rows": [
              ["Date", "Type", "Amount", "Description"]
            ],
            "rows": [
              {
                "row_index": 1,
                "cells": [
                  {"col_index": 1, "text": "2026-03-01", "bbox": [75, 205, 140, 220]},
                  {"col_index": 2, "text": "Repair", "bbox": [145, 205, 220, 220]},
                  {"col_index": 3, "text": "1250.00", "bbox": [225, 205, 300, 220]},
                  {"col_index": 4, "text": "Windshield replacement", "bbox": [305, 205, 520, 220]}
                ]
              }
            ],
            "column_headers": ["Date", "Type", "Amount", "Description"],
            "parser_confidence": 0.93
          }
        ],
        "forms": [
          {
            "form_region_id": "frm_1",
            "bbox": [70, 100, 320, 150],
            "fields": [
              {
                "field_name": "Customer Name",
                "field_value": "John Smith",
                "label_bbox": [72, 110, 170, 128],
                "value_bbox": [175, 110, 290, 128]
              },
              {
                "field_name": "Policy Number",
                "field_value": "PL-88291",
                "label_bbox": [72, 130, 170, 148],
                "value_bbox": [175, 130, 270, 148]
              }
            ]
          }
        ]
      }
    ],
    "document_level_features": {
      "repeated_headers": ["Claim Submission Form"],
      "repeated_footers": ["Page {n} of 3"],
      "language": "en"
    },
    "full_text": "Claim Submission Form\nCustomer Name: John Smith\nPolicy Number: PL-88291\n..."
  },
  "quality": {
    "status": "ok",
    "warnings": [],
    "errors": []
  }
}
```

## 3. DOCX normalized JSON schema (pre-segmentation)

### Design notes
DOCX usually has cleaner structure than PDF. Preserve:
- section hierarchy
- paragraphs
- runs/styles if useful
- tables
- headers/footers
- list structure

### Sample shape

```json
{
  "artifact_version": "1.0",
  "normalization_version": "1.0",
  "request_id": "req_12345",
  "document_id": "doc_docx_001",
  "source": {
    "s3_uri": "s3://bucket/requests/req_12345/raw/cover_letter.docx",
    "original_filename": "cover_letter.docx",
    "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "extension": "docx",
    "checksum_sha256": "def456"
  },
  "document_type": "docx",
  "parser": {
    "parser_name": "docx_parser_v1",
    "parser_version": "1.1.0",
    "parsed_at": "2026-03-20T14:15:00Z"
  },
  "document_stats": {
    "size_bytes": 228912,
    "section_count": 2,
    "paragraph_count": 18,
    "table_count": 1,
    "text_char_count": 6421,
    "ocr_applied": false
  },
  "content": {
    "headers": [
      {
        "header_id": "hdr_1",
        "section_index": 1,
        "text": "Confidential",
        "paragraph_refs": ["p_hdr_1"]
      }
    ],
    "footers": [
      {
        "footer_id": "ftr_1",
        "section_index": 1,
        "text": "Prepared for internal processing only",
        "paragraph_refs": ["p_ftr_1"]
      }
    ],
    "sections": [
      {
        "section_id": "sec_1",
        "section_index": 1,
        "title": "Request Summary",
        "elements_in_order": [
          "p_1",
          "p_2",
          "tbl_1",
          "p_3"
        ],
        "paragraphs": [
          {
            "paragraph_id": "p_1",
            "style_name": "Heading1",
            "text": "Request Summary",
            "runs": [
              {
                "text": "Request Summary",
                "bold": true,
                "italic": false,
                "underline": false
              }
            ],
            "list_info": null,
            "alignment": "left"
          },
          {
            "paragraph_id": "p_2",
            "style_name": "Normal",
            "text": "The customer is requesting a policy update due to a recent address change.",
            "runs": [
              {
                "text": "The customer is requesting a policy update due to a recent address change.",
                "bold": false,
                "italic": false,
                "underline": false
              }
            ],
            "list_info": null,
            "alignment": "left"
          },
          {
            "paragraph_id": "p_3",
            "style_name": "Normal",
            "text": "Please review the attached details and process accordingly.",
            "runs": [
              {
                "text": "Please review the attached details and process accordingly.",
                "bold": false,
                "italic": false,
                "underline": false
              }
            ],
            "list_info": null,
            "alignment": "left"
          }
        ],
        "tables": [
          {
            "table_id": "tbl_1",
            "row_count": 3,
            "col_count": 2,
            "rows": [
              {
                "row_index": 1,
                "cells": [
                  {"col_index": 1, "text": "Customer Name"},
                  {"col_index": 2, "text": "John Smith"}
                ]
              },
              {
                "row_index": 2,
                "cells": [
                  {"col_index": 1, "text": "Policy Number"},
                  {"col_index": 2, "text": "PL-88291"}
                ]
              },
              {
                "row_index": 3,
                "cells": [
                  {"col_index": 1, "text": "New Address"},
                  {"col_index": 2, "text": "123 Main Street, Toronto, ON"}
                ]
              }
            ],
            "is_key_value_like": true
          }
        ]
      }
    ],
    "document_properties": {
      "title": "Cover Letter",
      "author": "Operations Team",
      "created_at": "2026-03-18T10:00:00Z",
      "modified_at": "2026-03-19T12:00:00Z"
    },
    "full_text": "Confidential\nRequest Summary\nThe customer is requesting a policy update due to a recent address change.\n..."
  },
  "quality": {
    "status": "ok",
    "warnings": [],
    "errors": []
  }
}
```

## 4. XLSX normalized JSON schema (pre-segmentation)

### Design notes
For XLSX, keep the workbook structured. Preserve:
- workbook/sheet metadata
- used ranges
- detected tables
- row/column headers
- loose cell regions
- formulas optionally
- cell coordinates

Do not flatten workbook into plain text.

### Sample shape

```json
{
  "artifact_version": "1.0",
  "normalization_version": "1.0",
  "request_id": "req_12345",
  "document_id": "doc_xlsx_001",
  "source": {
    "s3_uri": "s3://bucket/requests/req_12345/raw/supporting_data.xlsx",
    "original_filename": "supporting_data.xlsx",
    "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "extension": "xlsx",
    "checksum_sha256": "ghi789"
  },
  "document_type": "xlsx",
  "parser": {
    "parser_name": "xlsx_parser_v1",
    "parser_version": "1.2.0",
    "parsed_at": "2026-03-20T14:18:00Z"
  },
  "document_stats": {
    "size_bytes": 482192,
    "sheet_count": 2,
    "table_count": 2,
    "text_char_count": 12044,
    "ocr_applied": false
  },
  "content": {
    "workbook": {
      "sheet_names_in_order": ["Customer Details", "Transactions"],
      "defined_names": [],
      "properties": {
        "created_at": "2026-03-15T09:30:00Z",
        "modified_at": "2026-03-19T16:11:00Z"
      }
    },
    "sheets": [
      {
        "sheet_id": "sh_1",
        "sheet_name": "Customer Details",
        "sheet_index": 1,
        "visibility": "visible",
        "used_range": "A1:B6",
        "dimensions": {
          "max_row": 6,
          "max_col": 2
        },
        "detected_regions": [
          {
            "region_id": "reg_1",
            "region_type": "kv_region",
            "range": "A1:B6"
          }
        ],
        "tables": [],
        "kv_regions": [
          {
            "region_id": "kv_1",
            "range": "A1:B6",
            "pairs": [
              {
                "key_cell": "A1",
                "value_cell": "B1",
                "key_text": "Customer Name",
                "value_text": "John Smith"
              },
              {
                "key_cell": "A2",
                "value_cell": "B2",
                "key_text": "Policy Number",
                "value_text": "PL-88291"
              },
              {
                "key_cell": "A3",
                "value_cell": "B3",
                "key_text": "Address",
                "value_text": "123 Main Street, Toronto, ON"
              }
            ]
          }
        ],
        "cells": [
          {
            "cell_ref": "A1",
            "row": 1,
            "col": 1,
            "value": "Customer Name",
            "display_value": "Customer Name",
            "data_type": "string",
            "formula": null,
            "style_ref": "style_1"
          },
          {
            "cell_ref": "B1",
            "row": 1,
            "col": 2,
            "value": "John Smith",
            "display_value": "John Smith",
            "data_type": "string",
            "formula": null,
            "style_ref": "style_2"
          }
        ]
      },
      {
        "sheet_id": "sh_2",
        "sheet_name": "Transactions",
        "sheet_index": 2,
        "visibility": "visible",
        "used_range": "A1:D101",
        "dimensions": {
          "max_row": 101,
          "max_col": 4
        },
        "detected_regions": [
          {
            "region_id": "reg_2",
            "region_type": "table",
            "range": "A1:D101"
          }
        ],
        "tables": [
          {
            "table_id": "tbl_1",
            "range": "A1:D101",
            "header_row_index": 1,
            "column_headers": ["Date", "Transaction ID", "Type", "Amount"],
            "rows": [
              {
                "row_index": 2,
                "cells": [
                  {"cell_ref": "A2", "text": "2026-03-01"},
                  {"cell_ref": "B2", "text": "TXN-001"},
                  {"cell_ref": "C2", "text": "Repair"},
                  {"cell_ref": "D2", "text": "1250.00"}
                ]
              },
              {
                "row_index": 3,
                "cells": [
                  {"cell_ref": "A3", "text": "2026-03-05"},
                  {"cell_ref": "B3", "text": "TXN-002"},
                  {"cell_ref": "C3", "text": "Inspection"},
                  {"cell_ref": "D3", "text": "300.00"}
                ]
              }
            ],
            "row_count": 100,
            "is_structured_table": true
          }
        ],
        "kv_regions": [],
        "cells_sample": [
          {
            "cell_ref": "A1",
            "row": 1,
            "col": 1,
            "value": "Date",
            "display_value": "Date",
            "data_type": "string",
            "formula": null
          },
          {
            "cell_ref": "D2",
            "row": 2,
            "col": 4,
            "value": 1250.0,
            "display_value": "1250.00",
            "data_type": "number",
            "formula": null
          }
        ]
      }
    ],
    "full_text_proxy": "Workbook with sheets: Customer Details, Transactions. Customer Details contains key-value fields. Transactions contains a tabular data range with headers Date, Transaction ID, Type, Amount."
  },
  "quality": {
    "status": "ok",
    "warnings": [],
    "errors": []
  }
}
```

## 5. EML normalized JSON schema (pre-segmentation)

### Design notes
For EML, preserve:
- headers
- recipients
- subject/date
- main body
- quoted thread
- attachment metadata
- text and html body if available

### Sample shape

```json
{
  "artifact_version": "1.0",
  "normalization_version": "1.0",
  "request_id": "req_12345",
  "document_id": "doc_eml_001",
  "source": {
    "s3_uri": "s3://bucket/requests/req_12345/raw/request_email.eml",
    "original_filename": "request_email.eml",
    "mime_type": "message/rfc822",
    "extension": "eml",
    "checksum_sha256": "jkl012"
  },
  "document_type": "eml",
  "parser": {
    "parser_name": "eml_parser_v1",
    "parser_version": "1.0.0",
    "parsed_at": "2026-03-20T14:20:00Z"
  },
  "document_stats": {
    "size_bytes": 89221,
    "attachment_count": 2,
    "text_char_count": 9432,
    "ocr_applied": false
  },
  "content": {
    "headers": {
      "message_id": "<abc123@example.com>",
      "subject": "Policy Update Request",
      "from": [
        {
          "name": "Jane Doe",
          "email": "jane.doe@example.com"
        }
      ],
      "to": [
        {
          "name": "Operations Team",
          "email": "ops@company.com"
        }
      ],
      "cc": [
        {
          "name": "Manager",
          "email": "manager@company.com"
        }
      ],
      "bcc": [],
      "reply_to": [],
      "sent_at": "2026-03-19T18:12:00Z"
    },
    "body": {
      "text_body": "Hello,\nPlease update my policy address effective immediately. My new address is 123 Main Street, Toronto, ON.\nThanks,\nJane",
      "html_body": "<html><body><p>Hello,</p><p>Please update my policy address effective immediately...</p></body></html>",
      "latest_message": {
        "text": "Hello,\nPlease update my policy address effective immediately. My new address is 123 Main Street, Toronto, ON.\nThanks,\nJane",
        "start_offset": 0,
        "end_offset": 128
      },
      "quoted_history": [
        {
          "message_index": 1,
          "text": "On Tue, Mar 18, 2026 at 9:12 AM Operations Team wrote:\nPlease provide proof of address.",
          "start_offset": 129,
          "end_offset": 230
        }
      ],
      "signature": {
        "text": "Jane Doe\nSenior Analyst\n555-123-4567",
        "detected": true
      }
    },
    "attachments": [
      {
        "attachment_id": "att_1",
        "filename": "proof_of_address.pdf",
        "mime_type": "application/pdf",
        "size_bytes": 212321,
        "content_id": null,
        "disposition": "attachment",
        "related_document_id": "doc_pdf_002"
      },
      {
        "attachment_id": "att_2",
        "filename": "details.xlsx",
        "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "size_bytes": 91222,
        "content_id": null,
        "disposition": "attachment",
        "related_document_id": "doc_xlsx_002"
      }
    ],
    "thread_features": {
      "has_quoted_history": true,
      "quoted_message_count": 1,
      "language": "en"
    },
    "full_text": "Subject: Policy Update Request\nFrom: Jane Doe <jane.doe@example.com>\nTo: Operations Team <ops@company.com>\nHello,\nPlease update my policy address..."
  },
  "quality": {
    "status": "ok",
    "warnings": [],
    "errors": []
  }
}
```

## 6. What should be common across all four?

To make downstream segmentation sane, try to normalize these concepts across formats:

### Common concepts
- `document_id`
- `document_type`
- `source`
- `parser`
- `document_stats`
- `content`
- `quality`

### Common useful sub-concepts
- ordered containers
- block-like units
- table-like units
- key-value-like units
- full-text or full-text proxy
- locators

That way the segmenter can say:
- if content has tables, inspect those
- if content has kv-like regions, consider those
- if content has ordered text blocks, group those

instead of needing completely unrelated logic per format.

## 7. Recommended JSON Schema style guidance

For implementation, I would strongly suggest:

### Use canonical internal contracts, not raw library output
Do not expose pdfplumber/docx/openpyxl/email parser raw structures directly.

Wrap them.

### Keep optional fields nullable
Different formats will not support the same richness.

### Prefer arrays with IDs over deeply nested anonymous blobs
This makes provenance and debugging easier.

### Preserve locators even if chunking does not use all of them at first
You will want them later.

### Include parser/version metadata in every artifact
This is essential for reprocessing and debugging.

## 8. A practical simplification

You asked for “schema,” but in implementation I would use two layers:

### Layer 1: Canonical artifact contract
Versioned internal JSON shape like the above.

### Layer 2: JSON Schema validation
A looser schema that validates:
- required top-level envelope fields
- content structure for the specific document type
- IDs and types

That keeps your contracts stable without making the schema impossibly rigid.

## 9. Recommended next step

The best next move is to define the **segment artifact schema** next, because it should be derived cleanly from these pre-segmentation normalized artifacts.

That would let you lock in:
- parser output contract
- segmenter input/output contract
- chunker input contract

as one consistent chain.

