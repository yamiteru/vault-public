# Allocators

| Name              | Allocation    | Deallocation  |
-----------------------------------------------------
| TLSF              | O(1)          | O(1)          |
| O1Heap            | O(1)          | O(1)          |
| RT-Mimalloc       | O(1)          | O(1)          |
| Half-fit          | O(1)          | O(1)          |
| Best-fit          | O(H/2M)       | O(1)          |
| First-fit         | O(H/2M)       | O(1)          |
| DLalloc           | O(H/M)        | O(1)          |
| Binary-buddy      | O(log2(H/M))  | O(log2(H/M))  |

---

| Name              | Allocation    | Deallocation  |
-----------------------------------------------------
| TLSF              | 170           | 194           |
| O1Heap            | 77            | 71            |
| RT-Mimalloc       | 170           | 104           |
| Half-fit          | 136           | 166           |
| Best-fit          | 98385         | 126           |
| First-fit         | 81995         | 126           |
| DLalloc           | 721108        | 83            |
| Binary-buddy      | 1403          | 1379          |

---

## TLSF

## O1Heap

## RT-Mimalloc

## Half-fit

## Best-fit

## First-fit

## DLalloc

## Binary-buddy

