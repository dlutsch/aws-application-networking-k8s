# RAM Service Network Discovery Bug Fix

## Problem Summary
Gateway API Controller fails to discover VPC Lattice Service Networks shared via AWS RAM, reporting "VPC Lattice Service Network not found" even when:
- Service network exists and is shared via RAM
- Controller has proper IAM permissions (vpc-lattice:* and ram:*)
- `DEFAULT_SERVICE_NETWORK` environment variable is set correctly
- RAM sharing is verified working with AWS CLI

## Root Cause Analysis - UPDATE: Configuration Issue Found!

**ACTUAL ROOT CAUSE**: The real issue was in the **terraform configuration**, not the controller code!

The terraform module (`otp-fixtures/modules/gateway_api_controller/locals.tf`) was **missing the `enableServiceNetworkOverride` helm value**, which meant the controller never enabled override mode and searched for a service network named after the Gateway name instead of using `DEFAULT_SERVICE_NETWORK`.

### Secondary Issue (Addressed in this PR)
File: `pkg/aws/services/vpclattice.go`
Function: `findServiceNetworkViaVPCAssociation()`

While the primary issue was configuration, this function also has a defensive coding issue:

```go
func (d *defaultLattice) FindServiceNetwork(ctx context.Context, nameOrId string) (*ServiceNetworkInfo, error) {
    // Step 1: Override nameOrId if ServiceNetworkOverrideMode is enabled
    if config.ServiceNetworkOverrideMode {
        nameOrId = config.DefaultServiceNetwork  // ✅ Uses DEFAULT_SERVICE_NETWORK
    }

    // Step 2: Try to find in local (owned) service networks
    input := &vpclattice.ListServiceNetworksInput{}
    allSn, err := d.ListServiceNetworksAsList(ctx, input)
    // ... searches for nameOrId in local networks ...

    // Step 3: If not found locally, try RAM-shared networks
    return d.findServiceNetworkViaVPCAssociation(ctx, nameOrId)  // ❌ STILL USES nameOrId!
}
```

**What Actually Happened**:
1. Gateway created with name "pr-36-tokenizer-gateway"
2. `FindServiceNetwork` called with `nameOrId = "pr-36-tokenizer-gateway"`
3. **BUT** `ServiceNetworkOverrideMode` was FALSE (not configured in terraform!)
4. Controller searches for service network named "pr-36-tokenizer-gateway"
5. **FAILS** - No service network with that name exists

**What Should Happen**:
1. Gateway created with name "pr-36-tokenizer-gateway"
2. `FindServiceNetwork` called with `nameOrId = "pr-36-tokenizer-gateway"`  
3. `ServiceNetworkOverrideMode` is TRUE → overrides to `config.DefaultServiceNetwork`
4. Controller searches for "sn-04f8437d5c6e026b0"
5. **SUCCESS** - Finds the RAM-shared service network

### The Defensive Coding Issue
Even with correct configuration, `findServiceNetworkViaVPCAssociation` lacks validation:

```go
func (d *defaultLattice) findServiceNetworkViaVPCAssociation(ctx context.Context, nameOrId string) (*ServiceNetworkInfo, error) {
    associations, err := d.ListServiceNetworkVpcAssociationsAsList(ctx,
        &vpclattice.ListServiceNetworkVpcAssociationsInput{
            VpcIdentifier: aws.String(config.VpcID),  // ❌ No validation if empty!
        })
```

If `config.VpcID` is not set, the function fails with unclear error messages.

## The Fixes

### Fix 1: Terraform Configuration (PRIMARY - In otp-fixtures repo)
**File**: `otp-fixtures/modules/gateway_api_controller/locals.tf`  
**Change**: Added missing helm value

```hcl
{
  name  = "enableServiceNetworkOverride"
  value = "true"
}
```

This was the actual blocker - without this, override mode was never enabled!

### Fix 2: Add VPC ID Validation (SECONDARY - This PR)
Add defensive validation in `findServiceNetworkViaVPCAssociation` to ensure `config.VpcID` is set:

```go
func (d *defaultLattice) findServiceNetworkViaVPCAssociation(ctx context.Context, nameOrId string) (*ServiceNetworkInfo, error) {
    // Validate that VPC ID is configured
    if config.VpcID == "" {
        return nil, fmt.Errorf("cannot discover RAM-shared service networks: CLUSTER_VPC_ID is not configured")
    }

    // List all VPC-to-Service Network associations for the controller's VPC
    associations, err := d.ListServiceNetworkVpcAssociationsAsList(ctx,
        &vpclattice.ListServiceNetworkVpcAssociationsInput{
            VpcIdentifier: aws.String(config.VpcID),
        })
    ...
}
```

### Why This Fix Matters
Even though the primary issue was configuration, this validation provides:
1. **Better error messages** when `CLUSTER_VPC_ID` is misconfigured
2. **Defensive programming** - fail fast with clear errors
3. **Easier debugging** for future issues

## Testing the Fix

### Before Fix
```bash
# Gateway Status
kubectl get gateway -n pr-36-tokenizer pr-36-tokenizer-gateway
# Status: Programmed=False, Message="VPC Lattice Service Network not found"

# Controller can't find the service network even though:
aws ram list-resources --resource-owner OTHER-ACCOUNTS --resource-type vpc-lattice:ServiceNetwork
# Shows: sn-04f8437d5c6e026b0 with status AVAILABLE
```

### After Fix
```bash
# Gateway Status should show:
kubectl get gateway -n pr-36-tokenizer pr-36-tokenizer-gateway
# Status: Programmed=True, Message="aws-service-network-arn: arn:aws:vpc-lattice:..."

# Controller should successfully discover RAM-shared service network
```

## Files Modified
1. `pkg/aws/services/vpclattice.go` - Add VPC ID validation and improved logging

## Impact

### Primary Issue (Configuration)
- **Priority**: Critical - Was the actual blocker
- **Resolution**: Fixed in otp-fixtures terraform module
- **Affects**: All Gateway API Controller deployments

### This PR (Validation Fix)
- **Priority**: Low - Defensive improvement
- **Benefit**: Better error messages and debugging
- **Risk**: None - Pure validation addition
- **Affects**: Future misconfigurations will be easier to diagnose

## Related Issues
- Relates to RAM-based service network discovery feature
- Part of cross-account VPC Lattice support

## Rollback Plan
If the fix causes issues, rollback is simple - revert the validation check. However, the fix is defensive and should not cause any regressions.
