# 2. Data Storage and File Systems

Data storage determines the physical layout of bytes on a disk and the network protocol used to access them. The transition from local file systems to distributed object storage defined the Big Data era.

!!! info "Historical Context: GFS and HDFS"
    In 2003, Google published the *Google File System (GFS)* paper. They had more web-crawl data than could fit on any single enterprise SAN, so they built an unreliable cluster of thousands of cheap hard drives and abstracted it using a master/slave architecture. In 2006, Doug Cutting at Yahoo! replicated GFS in Java, creating the **Hadoop Distributed File System (HDFS)**, launching the open-source Big Data revolution.

---

## 2.1 File Storage vs BLock Storage vs Object Storage

To a Data Engineer, the cloud abstracts the physical hard drives into three distinct tiers.

### Block Storage (EBS, local SSD)
Data is divided into raw, fixed-size blocks (e.g., 4KB sectors) managed by the OS (ext4). This is the absolute fastest storage and is required for databases like Postgres or Kafka brokers because you can overwrite just the 4 bytes holding a specific row without touching the rest of the disk.

### File Storage (NFS, EFS)
Data is accessed via a hierarchical tree (folders, files) using protocols like NFS or SMB. It's slower than Block, but allows multiple EC2 instances to natively mount the same `Z:\` drive simultaneously.

### Object Storage (Amazon S3, Azure Blob)
The foundation of modern Data Lakes. Data is flat; there are no folders. A "file" is an Object (data payload + boundless JSON metadata keys) accessed via HTTP REST APIs. It is perfectly elastic and infinitely cheaper because it is completely disconnected from Compute servers.

**Internal Insight:** Object storage is strictly immutable. You cannot `APPEND` to an object on S3. If you want to change the size of a 10MB Parquet file, you must completely rewrite the entire 10MB file.

---

## 2.2 Storage Tiers and Lifecycle

Because Object Storage is so vast, cloud providers use different hardware backends (SSDs vs Magnetic Tape) to optimize cost based on access frequency. Data Engineers must programmatically manage data lifecycles.

1. **Hot (S3 Standard):** Immediate millisecond access. Used for active ML model training or daily reporting. $23/TB.
2. **Warm (S3 Infrequent Access):** Older data queried occasionally. Storage is cheaper ($12/TB), but you are taxed heavily for fetching the data.
3. **Cold (S3 Glacier / Deep Archive):** Regulatory compliance data. $1/TB. Data is physically written to magnetic tape robots. Retrieval takes 12 to 48 hours.

??? example "Terraform Code: S3 Lifecycle Rules"
    Data Engineers use Infrastructure-as-Code to automate AWS shifting data from expensive NVMe drives to cheap magnetic tape.
    ```hcl
    resource "aws_s3_bucket_lifecycle_configuration" "lake_lifecycle" {
      bucket = aws_s3_bucket.data_lake.id

      rule {
        id     = "archive_old_financial_logs"
        status = "Enabled"

        filter {
          prefix = "raw/finance/"
        }

        # After 90 days, move to cheap Deep Archive (Magnetic Tape)
        transition {
          days          = 90
          storage_class = "DEEP_ARCHIVE"
        }
      }
    }
    ```

---

## 2.3 Hard Drive Physics (HDD vs SSD)

Understanding distributed storage requires understanding the limits of the physical hardware.

- **HDD (Hard Disk Drive):** A literal spinning magnetic platter with a robotic arm. It provides high sequential bandwidth (reading 10GB of Parquet sequentially is fast), but horrific random access (seeking 1,000 tiny JSON files forces the arm to physically move, causing latency).
- **SSD (Solid State Drive) / NVMe:** No moving parts. Floating-gate transistors. Supports massive parallel reads (IOPS). Modern NVMe drives bypass the SATA motherboard bottleneck to plug directly into the PCIe CPU lanes, enabling the Shared-Storage database architecture discussed in the Advanced section.

---

!!! abstract "References & Papers"
    - **Google File System (GFS)** (Ghemawat et al., SOSP 2003). The paper that proved commodity hardware clusters could outperform massive enterprise mainframes.
    - **Designing Data-Intensive Applications** - Chapter 3 (Storage and Retrieval) explains the physics of HDDs vs SSDs and how log-structured storage engines exploit sequential writes.
