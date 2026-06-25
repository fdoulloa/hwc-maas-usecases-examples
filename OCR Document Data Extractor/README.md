# 🤖 Demo — OCR Document Data Extractor

A Dify workflow that receives a file (image or document), sends it through an OCR API, and uses an LLM to extract structured data fields in JSON format.

## Overview

```
[Start] → [HTTP Request (OCR API)] → [LLM (DeepSeek)] → [End]
```

The user uploads a file, the workflow runs OCR on it, and the LLM extracts key fields returning clean JSON.

## Flow

| Step | Node | Description |
|------|------|-------------|
| 1 | **Start** | Receives a file input (`archivo`) — image or document |
| 2 | **HTTP Request** | Sends the file to `https://api.kromos.cl/ocr/api/v1/ocr` via `multipart/form-data` with `preprocess=true` |
| 3 | **LLM** | Uses DeepSeek Flash to extract fields from the OCR text and return structured JSON |
| 4 | **End** | Outputs the LLM text (JSON) |

## Output Format

The LLM returns a JSON object with the following fields:

```json
{
  "nombre": "string or null",
  "rut": "string or null",
  "fecha": "string or null",
  "monto": "string or null"
}
```

Missing fields are returned as `null`.

## Dependencies

| Plugin | Version |
|--------|---------|
| `langgenius/deepseek` | `0.0.17` |

## Configuration

| Parameter | Value |
|-----------|-------|
| OCR endpoint | `https://api.kromos.cl/ocr/api/v1/ocr` |
| OCR auth | `X-API-Key` header |
| LLM model | `deepseek-v4-flash` |
| LLM temperature | `0.7` |

> ⚠️ **Before importing:** Replace the `X-API-Key` value in the HTTP Request node with your own Kromos OCR API key.

## Import

1. In Dify, go to **Studio → Import DSL**
2. Upload `demo.yml`
3. Update the API key in the HTTP Request node
4. Run with any image or PDF containing name, RUT, date, or amount fields
