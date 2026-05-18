# File System

The Linux filesystem layer provides a common interface over different on-disk formats and storage backends.

## VFS

The Virtual Filesystem Switch (VFS) abstracts common objects:

- superblocks for mounted filesystems
- inodes for file metadata
- dentries for pathname lookup
- file objects for open file state

## Data Path

Typical file access involves:

1. Path lookup through dentries and inodes.
2. Permission and mount checks.
3. Reading from or writing to the page cache.
4. Dispatch to the underlying filesystem if storage access is needed.

## Operational Notes

- Metadata-heavy workloads often stress lookup and journaling paths.
- Small random writes can amplify writeback costs.
- Mount options materially change consistency and performance behavior.
