
# Low L-01 Title: `Stable2::calcReserveAtRatioLiquidity` and `Stable2::calcReserveAtRatioSwap` functions should revert when they don't converge

In case of `Stable2::calcReserveAtRatioLiquidity` and `Stable2::calcReserveAtRatioSwap` functions don't coverge to the proper result it should revert or it will return 0 value and it could affect the protocol.

