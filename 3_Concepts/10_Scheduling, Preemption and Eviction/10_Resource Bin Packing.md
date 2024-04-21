kube-scheduler의 `NodeResourceFit` plugin은 리소스의 bin packing을 지원하는 두 scoring 전략이 있다: `MostAllocated`와 `RequestedToCapacityRatio`

## Enabling bin packing using MostAllocated strategy
## Enabling bin packing using RequestedToCapacityRatio
### Tuning the score function
### Node scoring for capacity allocation