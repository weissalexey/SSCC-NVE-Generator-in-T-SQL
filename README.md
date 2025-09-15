# SSCC-NVE-Generator-in-T-SQL
Хорошо 👍 Ниже заготовка статьи для GitHub (можно оформить как README.md), где объясняется генерация SSCC/NVE в SQL с примером кода:

---

# SSCC (NVE) Generator in T-SQL

This project demonstrates how to generate **SSCC / NVE codes** in Microsoft SQL Server according to the GS1 standard.
It is designed for logistics environments (e.g. WinSped / XXANVE) where a new SSCC has to be created for each shipment.

---

## 📖 Background

An SSCC (Serial Shipping Container Code), also known as **NVE** in Germany, is an 18-digit number used to uniquely identify logistic units.

The structure is:

```
[Extension digit][Company Prefix][Serial Reference][Check digit]
```

* **Extension digit (1)** – free digit (0–9), used when serial range is exhausted.
* **Company Prefix (7–9 digits)** – derived from your GLN (without its check digit).
* **Serial Reference (up to 9 digits)** – sequential counter, unique within the prefix.
* **Check digit (1)** – calculated with GS1 Mod10 algorithm.

---

## ⚙️ SQL Script

```sql
-- Parameters
DECLARE @GLN           CHAR(13) = '4260708480009';
DECLARE @BaseLength    INT = 9;      -- company prefix length (7/8/9)
DECLARE @Extension     CHAR(1) = '3';-- extension digit

-- Extract company prefix
DECLARE @Core12 CHAR(12) = LEFT(@GLN, 12);
DECLARE @CompanyPrefix VARCHAR(12) = LEFT(@Core12, @BaseLength);

-- Get last serial from XXANVE
DECLARE @LastSerial BIGINT;
SELECT TOP 1 @LastSerial = MAX(NVEIntNr) FROM XXANVE;
IF @LastSerial IS NULL SET @LastSerial = 0;
DECLARE @NewSerial BIGINT = @LastSerial + 1;

-- Serial length
DECLARE @SerialLen INT = 17 - (1 + LEN(@CompanyPrefix));

-- Build 17 digits (without check digit)
DECLARE @Body17 VARCHAR(17) =
    @Extension + @CompanyPrefix +
    RIGHT(REPLICATE('0', @SerialLen) + CAST(@NewSerial AS VARCHAR), @SerialLen);

-- Calculate GS1 check digit (mod 10)
DECLARE @i INT = 1, @Sum INT = 0, @Weight INT, @Digit INT;
WHILE @i <= 17
BEGIN
    SET @Digit = CAST(SUBSTRING(REVERSE(@Body17), @i, 1) AS INT);
    SET @Weight = CASE WHEN @i % 2 = 1 THEN 3 ELSE 1 END;
    SET @Sum += @Digit * @Weight;
    SET @i += 1;
END
DECLARE @Check CHAR(1) = CAST((10 - (@Sum % 10)) % 10 AS CHAR(1));

-- Final SSCC/NVE
DECLARE @SSCC CHAR(18) = @Body17 + @Check;
DECLARE @AI00 CHAR(20) = '00' + @SSCC;

-- Result
SELECT 
    @LastSerial     AS LastSerial,
    @NewSerial      AS NewSerial,
    @CompanyPrefix  AS CompanyPrefix,
    @SSCC           AS NewSSCC,
    @AI00           AS AI00_SSCC;
```

---

## 🔍 How it works

1. Takes your **GLN** (13 digits).
2. Removes the GLN check digit → takes first 7–9 digits as the **Company Prefix**.
3. Reads the last used serial number from `XXANVE.NVEIntNr`.
4. Increments the serial (`+1`).
5. Builds the 17-digit body: `[Extension + CompanyPrefix + SerialRef]`.
6. Computes the **check digit** (GS1 Mod10 algorithm).
7. Returns the final **SSCC** and **AI(00)+SSCC** format.

---

## 🚀 Example Output

For GLN `4260708480009`, Extension `3`, and Serial `2851336`,
the generated code will be:

```
SSCC:     342607084828513360
AI(00):   00342607084828513360
```

---

## 📦 Extensions

* Generate **batch of SSCCs** (e.g. TOP 100) by wrapping the logic in a loop/CTE.
* Insert directly into `XXANVE` with `INSERT ... OUTPUT`.
* Create a **stored procedure** `dbo.GenerateNVE` with parameters (`@Count`, `@GLN`, `@Extension`).
* Export as **GS1-128 barcode** for printing labels.

---

👉 Would you like me to extend this into a **ready-to-use stored procedure** (`sp_GenerateNVE`) so you can call it like `EXEC sp_GenerateNVE @Count=100` and get back a table of new NVEs?
****
