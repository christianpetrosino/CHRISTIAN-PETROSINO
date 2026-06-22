[Petrosino_Christian_SQL.sql](https://github.com/user-attachments/files/29218595/Petrosino_Christian_SQL.sql)

-- ===========================================
-- TOYSGROUP CASE STUDY
-- Soluzione completa Task 1-9 (MySQL)
-- ===========================================

DROP DATABASE IF EXISTS ToysGroupDB;
CREATE DATABASE ToysGroupDB;
USE ToysGroupDB;

-- ======================
-- TASK 1-2: DDL
-- ======================

CREATE TABLE Category (
    CategoryID INT PRIMARY KEY,
    CategoryName VARCHAR(50) NOT NULL
);

CREATE TABLE Product (
    ProductID INT PRIMARY KEY,
    ProductName VARCHAR(100) NOT NULL,
    UnitPrice DECIMAL(10,2),
    CategoryID INT NOT NULL,
    FOREIGN KEY (CategoryID) REFERENCES Category(CategoryID)
);

CREATE TABLE SalesRegion (
    RegionID INT PRIMARY KEY,
    RegionName VARCHAR(50) NOT NULL
);

CREATE TABLE State (
    StateID INT PRIMARY KEY,
    StateName VARCHAR(50) NOT NULL,
    RegionID INT NOT NULL,
    FOREIGN KEY (RegionID) REFERENCES SalesRegion(RegionID)
);

CREATE TABLE Sales (
    SaleID INT PRIMARY KEY,
    SaleDate DATE NOT NULL,
    Quantity INT NOT NULL,
    Amount DECIMAL(10,2) NOT NULL,
    ProductID INT NOT NULL,
    StateID INT NOT NULL,
    FOREIGN KEY (ProductID) REFERENCES Product(ProductID),
    FOREIGN KEY (StateID) REFERENCES State(StateID)
);

-- ======================
-- TASK 3: INSERT DATI
-- ======================

INSERT INTO Category VALUES
(1,'Bikes'),
(2,'Clothing');

INSERT INTO Product VALUES
(1,'Bikes-100',500,1),
(2,'Bikes-200',700,1),
(3,'Bike Gloves M',20,2),
(4,'Bike Gloves L',20,2);

INSERT INTO SalesRegion VALUES
(1,'WestEurope'),
(2,'SouthEurope');

INSERT INTO State VALUES
(1,'France',1),
(2,'Germany',1),
(3,'Italy',2),
(4,'Greece',2);

INSERT INTO Sales VALUES
(1,'2024-01-10',2,1000,1,1),
(2,'2024-01-12',1,700,2,2),
(3,'2024-01-15',5,100,3,3),
(4,'2024-01-20',3,60,4,4);

-- ======================
-- TASK 4.1 PK UNIVOCHE
-- ======================

SELECT CategoryID, COUNT(*) FROM Category GROUP BY CategoryID HAVING COUNT(*) > 1;
SELECT ProductID, COUNT(*) FROM Product GROUP BY ProductID HAVING COUNT(*) > 1;
SELECT RegionID, COUNT(*) FROM SalesRegion GROUP BY RegionID HAVING COUNT(*) > 1;
SELECT StateID, COUNT(*) FROM State GROUP BY StateID HAVING COUNT(*) > 1;
SELECT SaleID, COUNT(*) FROM Sales GROUP BY SaleID HAVING COUNT(*) > 1;

-- ======================
-- TASK 4.2 ELENCO TRANSAZIONI
-- ======================

SELECT
    s.SaleID AS CodiceDocumento,
    s.SaleDate AS DataVendita,
    p.ProductName AS Prodotto,
    c.CategoryName AS Categoria,
    st.StateName AS Stato,
    r.RegionName AS RegioneVendita,
    CASE
        WHEN DATEDIFF(CURDATE(), s.SaleDate) > 180 THEN 'True'
        ELSE 'False'
    END AS Oltre180Giorni
FROM Sales s
JOIN Product p ON s.ProductID = p.ProductID
JOIN Category c ON p.CategoryID = c.CategoryID
JOIN State st ON s.StateID = st.StateID
JOIN SalesRegion r ON st.RegionID = r.RegionID;

-- ======================
-- TASK 4.3 PRODOTTI SOPRA MEDIA
-- ======================

SELECT
    ProductID,
    SUM(Quantity) AS TotaleVenduto
FROM Sales
WHERE SaleDate >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
GROUP BY ProductID
HAVING SUM(Quantity) >
(
    SELECT AVG(TotQta)
    FROM (
        SELECT SUM(Quantity) AS TotQta
        FROM Sales
        WHERE SaleDate >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
        GROUP BY ProductID
    ) T
);

-- ======================
-- TASK 4.4 FATTURATO PRODOTTI
-- ======================

SELECT
    p.ProductID,
    p.ProductName,
    YEAR(s.SaleDate) AS Anno,
    SUM(s.Amount) AS FatturatoTotale
FROM Product p
JOIN Sales s ON p.ProductID = s.ProductID
GROUP BY p.ProductID, p.ProductName, YEAR(s.SaleDate);

-- ======================
-- TASK 4.5 FATTURATO PER STATO
-- ======================

SELECT
    st.StateName,
    YEAR(s.SaleDate) AS Anno,
    SUM(s.Amount) AS FatturatoTotale
FROM Sales s
JOIN State st ON s.StateID = st.StateID
GROUP BY st.StateName, YEAR(s.SaleDate)
ORDER BY Anno DESC, FatturatoTotale DESC;

-- ======================
-- TASK 4.6 CATEGORIA PIU' RICHIESTA
-- ======================

SELECT
    c.CategoryName,
    SUM(s.Quantity) AS QuantitaVenduta
FROM Sales s
JOIN Product p ON s.ProductID = p.ProductID
JOIN Category c ON p.CategoryID = c.CategoryID
GROUP BY c.CategoryName
ORDER BY QuantitaVenduta DESC
LIMIT 1;

-- ======================
-- TASK 4.7 PRODOTTI INVENDUTI
-- ======================

SELECT p.ProductID, p.ProductName
FROM Product p
LEFT JOIN Sales s ON p.ProductID = s.ProductID
WHERE s.ProductID IS NULL;

SELECT p.ProductID, p.ProductName
FROM Product p
WHERE NOT EXISTS (
    SELECT 1
    FROM Sales s
    WHERE s.ProductID = p.ProductID
);

-- ======================
-- TASK 4.8 VIEW PRODOTTI
-- ======================

CREATE VIEW vw_ProductInfo AS
SELECT
    p.ProductID,
    p.ProductName,
    c.CategoryName
FROM Product p
JOIN Category c ON p.CategoryID = c.CategoryID;

-- ======================
-- TASK 4.9 VIEW GEOGRAFICA
-- ======================

CREATE VIEW vw_Geography AS
SELECT
    st.StateID,
    st.StateName,
    r.RegionID,
    r.RegionName
FROM State st
JOIN SalesRegion r ON st.RegionID = r.RegionID;

