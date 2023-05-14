# Chapter 21

## Intro

- So far, we have a big assumption to relax:
  - **all** pages reside in physical **memory**
- However, not with large address spaces, the OS will need a place to stash away portion of large address spaces that are not in great demand
- Without it, the application programmer will need to **manually** move in and out of memory
- In the modern system, this place is usually the **hard disk drive**

## Swap Space

- Swap space is the space on the disk for moving pages back and forth
- The size of the swap space determines how many memory pages ew can use in a system
- Swap space is not the only on-disk location for swapping demand
  - For example, if a page is a code page and found in the physical memory but we need some room for other usages,
  - we can use its memory space for other demand since we can swap it in again from the on-disk binary in the file system

## The Present Bit

- It is the bit in the page table entry to show that if the current page is still in the physical memory
- Or it is already swapped into the hard disk
- If the present bit is set to 0, the act of accessing the page is called a **page fault**

## Page Fault

- In either hardware-managed / software-managed systems, it is the OS to handle the page fault
- To handle the page fault, the OS will swap the page into memory as the following
  1.  Update the page table
  2.  Mark the page as present
  3.  Update the PFN
  4.  Retry the instruction
  5.  TLB miss... and the rest is the same (update TLB, retry)

### What If Memory Is Full?

- The OS will first **swap out** one or more pages before swapping in the target page
- The process of picking a page to swap out is called **page-replacement policy**

### Page-Fault Control Flow Algorithm

- Hardware

```
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True)   // TLB Hit
  if (CanAccess(TlbEntry.ProtectBits) == True)
    Offset = VirtualAddress & OFFSET_MASK
    PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
    Register = AccessMemory(PhysAddr)
  else
      RaiseException(PROTECTION_FAULT)
else
  PTEAddr = PTBR + (VPN * sizeof(PTE))
  PTE = AccessMemory(PTEAddr)
  if (PTE.Valid == False)
    RaiseException(SEGMENTATION_FAULT)
  else
    if (CanAccess(PTE.ProtectBits) == False)
      RaiseException(PROTECTION_FAULT)
    else if (PTE.Present == True)
        // assuming hardware-managed TLB
        TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
        RetryInstruction()
    else if (PTE.Present == False)
        RaiseException(PAGE_FAULT)
```

- Software

```
PFN = FindFreePhysicalPage()
if (PFN == -1)              // no free page found
    PFN = EvictPage()       // run replacement algorithm
DiskRead(PTE.DiskAddr, PFN) // sleep (waiting for I/O)
PTE.present = True          // update page table with present
PTE.PFN = PFN               // bit and translation (PFN)
RetryInstruction()          // retry instruction
```

### Swap Daemon (Page Daemon)

- The background thread responsible for freeing memory
- With two thresholds, **high watermark** (HW) and **low watermark** (LW)
- when the OS sees the number of free pages are less than HW, it will wake up the swap daemon
- the swap daemon will keep swapping pages out of memory to the disk until there are at least HW free pages
- In the algorithm above, the OS checks if the free pages are enough
- If not, the OS will inform the background paging
