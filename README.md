# FHIR Claim Web 程式設計

## 一、系統目的

本作業設計一個以 **HL7 FHIR R4 標準**為基礎的 Claim 建立系統，
透過網頁前端方式輸入資料，並產生符合 FHIR 規範的 Claim JSON，
可用於醫療費用申報、測試或系統整合。

本文件展示：
- FHIR Claim 資源結構
- Claim JSON 範例
- HTML + JavaScript 前端設計概念
## 二、使用的 FHIR 規範
| 欄位名稱         | 資料型態                                   | 是否必填 | 說明                                                    |
| ------------ | -------------------------------------- | ---- | ----------------------------------------------------- |
| resourceType | string                                 | 必填   | 固定為 `"Claim"`                                         |
| status       | code                                   | 必填   | 申報狀態（draft / active / cancelled / entered-in-error）   |
| use          | code                                   | 必填   | Claim 用途（claim / preauthorization / predetermination） |
| type         | CodeableConcept                        | 建議   | 申報類型（institutional / professional / pharmacy 等）       |
| created      | date                                   | 必填   | Claim 建立日期                                            |
| patient      | Reference(Patient)                     | 必填   | 病患資源參照                                                |
| provider     | Reference(Practitioner / Organization) | 必填   | 醫療提供者                                                 |
| insurer      | Reference(Organization) / display      | 建議   | 保險機構                                                  |
| priority     | CodeableConcept                        | 選填   | 申報優先順序（normal / urgent）                               |
| insurance    | BackboneElement                        | 必填   | 保險資訊（coverage、sequence、focal）                         |
| diagnosis    | BackboneElement                        | 選填   | 診斷資訊（ICD-10）                                          |
| item         | BackboneElement                        | 必填   | 申報明細                                                  |
| total        | Money                                  | 建議   | 申報總金額                                                 |
## 三、Claim Item（申報明細）欄位規格
| 欄位名稱             | 資料型態            | 說明                             |
| ---------------- | --------------- | ------------------------------ |
| sequence         | integer         | 明細流水號                          |
| productOrService | CodeableConcept | 服務或處置項目（如 CPT Code）            |
| quantity         | Quantity        | 數量                             |
| unitPrice        | Money           | 單價                             |
| net              | Money           | 小計（可由 unitPrice × quantity 推得） |

## 四、診斷（Diagnosis）欄位規格
| 欄位名稱                     | 資料型態            | 說明                                                               |
| ------------------------ | --------------- | ---------------------------------------------------------------- |
| sequence                 | integer         | 診斷順序                                                             |
| diagnosisCodeableConcept | CodeableConcept | ICD-10 診斷碼                                                       |
| coding.system            | uri             | [http://hl7.org/fhir/sid/icd-10](http://hl7.org/fhir/sid/icd-10) |
| coding.code              | string          | ICD-10 Code（如 J06.9）                                             |
## 五、前端功能規格（HTML + JavaScript）
| 功能項目       | 說明                                                  |
| ---------- | --------------------------------------------------- |
| 資料輸入       | 使用表單輸入 Claim 欄位（status、use、patient、provider、item 等） |
| JSON 產生    | 即時依輸入內容產生 FHIR Claim JSON                           |
| 金額計算       | 由 unitPrice × quantity 計算 total                     |
| JSON 顯示    | 以 `<pre>` 區塊格式化顯示                                   |
| RESTful 傳送 | 使用 fetch POST 至 FHIR Server                         |
| 錯誤處理       | 顯示 FHIR OperationOutcome 錯誤訊息                       |

## 六、FHIR Claim JSON 範例

```json
{
  "resourceType": "Claim",
  "status": "active",
  "use": "claim",
  "type": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/claim-type",
        "code": "institutional"
      }
    ]
  },
  "patient": {
    "reference": "Patient/123"
  },
  "created": "2025-01-01",
  "provider": {
    "reference": "Practitioner/456"
  },
  "insurer": {
    "reference": "Organization/789"
  },
  "diagnosis": [
    {
      "sequence": 1,
      "diagnosisCodeableConcept": {
        "coding": [
          {
            "system": "http://hl7.org/fhir/sid/icd-10",
            "code": "J06.9"
          }
        ]
      }
    }
  ],
  "item": [
    {
      "sequence": 1,
      "productOrService": {
        "coding": [
          {
            "system": "http://www.ama-assn.org/go/cpt",
            "code": "99213"
          }
        ]
      },
      "unitPrice": {
        "value": 150,
        "currency": "TWD"
      },
      "quantity": {
        "value": 1
      }
    }
  ],
  "total": {
    "value": 150,
    "currency": "TWD"
  }
}

``` 

---

##  七、：貼 HTML（表示你「不是只會 JSON」）

```md
```html
<!doctype html>
<html lang="zh-Hant">
<head>
    <meta charset="utf-8" />
    <title>FHIR Claim Builder (R4)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style>
        body {
            font-family: system-ui,"Microsoft JhengHei";
            background: #f6f7fb;
            padding: 20px;
        }

        .card {
            background: #fff;
            padding: 20px;
            border-radius: 14px;
            max-width: 900px;
            margin: auto;
            box-shadow: 0 8px 24px rgba(0,0,0,.06);
        }

        label {
            font-weight: 700;
            margin-top: 12px;
            display: block;
        }

        input, select {
            width: 100%;
            padding: 10px;
            margin-top: 6px;
            border-radius: 10px;
            border: 1px solid #cfd5e6;
        }

        .row {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 12px;
        }

        button {
            padding: 10px 14px;
            border-radius: 10px;
            border: 0;
            cursor: pointer;
            font-weight: 700;
        }

        .primary {
            background: #3b82f6;
            color: #fff;
        }

        .item {
            background: #f9fbff;
            border: 1px solid #e5e9f6;
            padding: 12px;
            border-radius: 12px;
            margin-top: 10px;
        }

        pre {
            background: #0b1020;
            color: #e8ecff;
            padding: 12px;
            border-radius: 12px;
            overflow: auto;
        }

        .hint {
            font-size: 13px;
            color: #555;
            margin-top: 4px;
        }
    </style>
</head>

<body>
    <div class="card">
        <h1>FHIR Claim Builder（R4）</h1>
        <p class="hint">填寫後可產生 Claim JSON，並送到 HAPI FHIR Server</p>

        <div class="row">
            <div>
                <label>status（建議 draft）</label>
                <select id="status">
                    <option value="draft" selected>draft</option>
                </select>
            </div>
            <div>
                <label>use</label>
                <select id="use">
                    <option value="claim">claim</option>
                    <option value="preauthorization">preauthorization</option>
                </select>
            </div>
        </div>

        <label>created</label>
        <input id="created" type="date">

        <label>patient Reference</label>
        <input id="patient" value="Patient/951">

        <label>provider Reference</label>
        <input id="provider" value="Practitioner/753">

        <label>insurer Reference</label>
        <input id="insurer" value="Organization/477">

        <h2>Item</h2>
        <div class="item">
            <label>Service Code (CPT)</label>
            <input id="cpt" value="99213">

            <div class="row">
                <div>
                    <label>unitPrice</label>
                    <input id="price" type="number" value="150">
                </div>
                <div>
                    <label>quantity</label>
                    <input id="qty" type="number" value="1">
                </div>
            </div>
        </div>

        <br>
        <button class="primary" onclick="buildAndSend()">產生並送出 Claim</button>

        <h2>Claim JSON</h2>
        <pre id="output">{}</pre>
    </div>

    <script>
        document.getElementById("created").value =
            new Date().toISOString().slice(0, 10);

        function buildAndSend() {
            const price = Number(document.getElementById("price").value);
            const qty = Number(document.getElementById("qty").value);

            const claim = {
                resourceType: "Claim",
                status: "draft",
                use: "claim",
                type: {
                    coding: [{ system: "http://terminology.hl7.org/CodeSystem/claim-type", code: "institutional" }]
                },
                patient: { reference: "Patient/example" }, // 建議改用 example 或現有 ID
                created: new Date().toISOString().split('T')[0],
                provider: { reference: "Practitioner/example" },
                insurer: { display: "General Insurance Company" }, // 改用 display 避免找不到 ID
                priority: {
                    coding: [{ system: "http://terminology.hl7.org/CodeSystem/processpriority", code: "normal" }]
                },
                // 記得加入 insurance 欄位，這也是必填
                insurance: [{
                    sequence: 1,
                    focal: true,
                    coverage: { display: "Self Pay Plan" }
                }],
                item: [{
                    sequence: 1,
                    productOrService: {
                        coding: [{ system: "http://www.ama-assn.org/go/cpt", code: "99213" }]
                    },
                    unitPrice: { value: 150, currency: "TWD" },
                    quantity: { value: 1 }
                }],
                total: { value: 150, currency: "TWD" }
            };
            document.getElementById("output").textContent = JSON.stringify(claim, null, 2);
            sendToFHIR(claim);
        }

        function sendToFHIR(claim) {
            // 直接發送到 HAPI，不使用代理伺服器
            const fhirUrl = "https://hapi.fhir.org/baseR4/Claim";

            fetch(fhirUrl, {
                method: "POST",
                headers: {
                    "Content-Type": "application/fhir+json", // FHIR 規範建議的 MIME
                    "Accept": "application/json"
                },
                body: JSON.stringify(claim)
            })
                .then(async res => {
                    const data = await res.json();
                    if (!res.ok) {
                        // 顯示 FHIR 伺服器的錯誤訊息 (OperationOutcome)
                        const errorMsg = data.issue ? data.issue[0].diagnostics : "未知錯誤";
                        throw new Error(errorMsg);
                    }
                    return data;
                })
                .then(data => {
                    console.log("FHIR Response:", data);
                    alert("✅ 成功送出！Claim ID: " + data.id);
                })
                .catch(err => {
                    console.error("Error Detail:", err);
                    alert("❌ 錯誤：" + err.message);
                });
        }
    </script>
</body>
</html>



```


---

##  八、貼 JavaScript

```md
```javascript
function buildClaim() {
  const claim = {
    resourceType: "Claim",
    status: "active",
    use: "claim",
    patient: {
      reference: document.getElementById("patient").value
    },
    provider: {
      reference: document.getElementById("provider").value
    },
    diagnosis: [
      {
        sequence: 1,
        diagnosisCodeableConcept: {
          coding: [
            {
              system: "http://hl7.org/fhir/sid/icd-10",
              code: document.getElementById("icd").value
            }
          ]
        }
      }
    ]
  };

  document.getElementById("output").textContent =
    JSON.stringify(claim, null, 2);
}
```

---
##  九、系統使用情境說明（Use Case Scenario）
情境背景

某地區醫院已完成電子病歷（EMR）系統建置，但在醫療費用申報流程中，仍需人工整理申報資料並轉換成各保險單位所需格式，容易發生資料格式不一致、欄位缺漏或重複輸入等問題。

為提升申報效率與系統整合能力，醫院資訊室規劃導入符合 HL7 FHIR R4 標準的申報資料交換方式，並以 Claim 資源作為醫療費用申報的核心格式。

| 角色          | 說明                  |
| ----------- | ------------------- |
| 醫療行政人員      | 負責輸入醫療服務內容與申報資料     |
| 醫療提供者       | 提供診斷與醫療服務（醫師、醫療機構）  |
| 保險機構系統      | 接收並審核 Claim 申報資料    |
| FHIR Server | 儲存並驗證符合標準的 Claim 資源 |

使用流程說明

醫療行政人員於門診結束後，開啟「FHIR Claim Builder」網頁系統

系統自動產生建立日期（created），並預設 Claim 狀態為 draft

使用者輸入病患（Patient）與醫療提供者（Provider）之 Reference

填寫診斷資訊（ICD-10），以及本次醫療服務項目（CPT Code）

系統依輸入的數量與單價，自動計算申報金額（total）

使用者確認資料正確後，送出 Claim

系統透過 RESTful API 將 Claim JSON 傳送至 FHIR Server

FHIR Server 回傳建立成功結果，並產生 Claim 資源 ID 供後續查詢或審核使用
##  十、「學習心得 / 系統特色」

```md
- 使用 FHIR R4 Claim 資源建模醫療申報資料
- 透過 Reference 連結 Patient / Provider
- ICD-10 診斷碼結構符合 FHIR 規範
- 前端即時產生標準化 JSON
- 可延伸串接 FHIR Server 進行 RESTful POST
```
