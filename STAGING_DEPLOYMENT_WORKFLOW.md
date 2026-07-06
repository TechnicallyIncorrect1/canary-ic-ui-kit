# Staging Deployment Workflow for Dependency Upgrades

## Overview
This document outlines the complete staging deployment process for the three critical PRs with structured gates, validation steps, and rollback procedures.

---

## 1. PRE-DEPLOYMENT VALIDATION

### 1.1 Environment Checklist (Day 1)

```bash
# Validate staging infrastructure
./scripts/pre-deployment-check.sh

# Expected outputs:
# ✓ Staging database: Connected
# ✓ Staging CDN: Operational
# ✓ Monitoring: Active
# ✓ Logging: Operational
# ✓ Backups: Recent (< 24h)
```

**Validation Steps:**
```bash
# 1. Check disk space
df -h /staging | grep -v Use%
# Expect: >50GB available

# 2. Check memory
free -h | grep Mem
# Expect: >8GB available

# 3. Check services
systemctl status staging-api staging-web nginx postgres redis
# Expect: All active (running)

# 4. Check database
psql -h staging-db.internal postgres -c "SELECT version();"
# Expect: PostgreSQL connection successful

# 5. Verify backups
aws s3 ls s3://backups/staging/ --recursive | tail -5
# Expect: Recent backups present
```

---

## 2. DEPLOYMENT PHASES

### Phase 1: ic-ui-kit PR #6 (Next.js Security) - FAST TRACK
**Duration:** 2-4 hours | **Risk:** 🟢 LOW | **Rollback Time:** 5 minutes

#### Stage 1A: Pre-Deployment (30 min)
```bash
# 1. Create feature branch for staging
git checkout -b staging/ic-ui-kit-pr6
git pull origin pull/6/head

# 2. Run all tests
npm ci
npm run test
npm run test:integration

# Expected output: All tests pass

# 3. Build artifact
npm run build
# Expected: Build succeeds, no warnings about next.js

# 4. Create staging Docker image
docker build -t staging-ic-ui-kit:pr6 .
docker tag staging-ic-ui-kit:pr6 staging-registry.internal/ic-ui-kit:pr6-$(date +%s)

# 5. Run security scan
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image staging-ic-ui-kit:pr6
# Expected: No HIGH/CRITICAL vulnerabilities
```

#### Stage 1B: Staging Deployment (1 hour)
```bash
# 1. Blue-Green Setup
# Current (Blue): ic-ui-kit:latest
# New (Green): ic-ui-kit:pr6

# Deploy to green environment
kubectl set image deployment/ic-ui-kit-green \
  ic-ui-kit=staging-registry.internal/ic-ui-kit:pr6-$(date +%s) \
  --namespace=staging

# Wait for rollout
kubectl rollout status deployment/ic-ui-kit-green -n staging

# 2. Health Checks
./scripts/staging-health-check.sh ic-ui-kit-green

# Expected checks:
# ✓ Pod status: Running
# ✓ Ready replicas: 3/3
# ✓ HTTP 200 on /api/health
# ✓ Next.js routes respond
# ✓ Static assets serve
```

#### Stage 1C: Validation (1-2 hours)
```bash
# 1. Security-specific tests
npm run test:security

# 2. Next.js specific tests
npm run test:nextjs

# 3. Smoke tests
npm run test:smoke -- --environment=staging

# 4. Performance baseline
npm run test:performance -- --baseline

# 5. Manual verification
curl -I https://staging-ic-ui-kit.internal/
# Expected: HTTP 200

# 6. Check logs for errors
kubectl logs -l app=ic-ui-kit-green -n staging --tail=100 | grep -i error
# Expected: No Next.js errors
```

#### Stage 1D: Traffic Switch (30 min)
```bash
# If all validations pass:
# Switch 10% traffic to green
kubectl patch service ic-ui-kit -n staging -p \
  '{"spec":{"selector":{"version":"pr6"}}}'

# Monitor for 15 minutes
for i in {1..15}; do
  echo "Monitor interval $i..."
  ./scripts/staging-health-check.sh
  sleep 60
done

# If all good, switch 100%
kubectl patch service ic-ui-kit -n staging -p \
  '{"spec":{"selector":{"version":"pr6-full"}}}'

# 7. Remove blue environment
kubectl delete deployment ic-ui-kit-blue -n staging
```

#### Stage 1E: Post-Deployment (30 min)
```bash
# 1. Document deployment
cat > deployments/ic-ui-kit-pr6-deployment.log <<EOF
Deploy: ic-ui-kit PR #6 (Next.js 15.5.18)
Time: $(date)
Status: SUCCESS
Replicas: 3/3 running
Health: All checks passed
EOF

# 2. Notify team
slack send "#deployments" "@channel IC-UI-Kit PR#6 deployed to staging. Monitoring for next 24h."

# 3. Schedule sign-off review
./scripts/create-review-task.sh pr6 24h
```

**Rollback if Issues Found:**
```bash
# Immediate rollback (< 5 minutes)
kubectl set image deployment/ic-ui-kit-green \
  ic-ui-kit=staging-registry.internal/ic-ui-kit:latest \
  --namespace=staging

kubectl rollout undo deployment/ic-ui-kit -n staging
systemctl restart ic-ui-kit-nginx
```

---

### Phase 2: canary-ic-ui-kit PR #9 (Axios Breaking Changes) - CAREFUL TRACK
**Duration:** 2-3 weeks | **Risk:** 🟡 MEDIUM | **Rollback Time:** 15 minutes

#### Stage 2A: Pre-Deployment Week 1 (Days 1-5)

**Day 1: Test Preparation**
```bash
# 1. Merge PR #9 into staging branch
git checkout -b staging/canary-pr9
git pull origin pull/9/head

# 2. Identify all HTTP client usages
grep -r "axios\|fetch\|http\." src/ --include="*.js" --include="*.ts" | wc -l
# Document count for regression testing

# 3. Run existing test suite
npm ci
npm run test 2>&1 | tee test-baseline.log

# Expected: Baseline pass rate documented
# Expected: Any failures marked as "expected due to breaking changes"

# 4. Create test environment
docker-compose -f docker-compose.staging.yml up -d
./scripts/wait-for-services.sh

# 5. Setup monitoring
./scripts/setup-monitoring.sh staging-pr9
```

**Day 2-3: Unit Test Execution**
```bash
# Run breaking change tests from TESTING_PLAN_AXIOS_1_16_0.md
npm run test -- tests/axios-1.16.0/fetch-adapter-limits.test.js
npm run test -- tests/axios-1.16.0/proxy-host-header.test.js
npm run test -- tests/axios-1.16.0/url-auth-decoding.test.js
npm run test -- tests/axios-1.16.0/parse-protocol.test.js
npm run test -- tests/axios-1.16.0/utf8-encoding.test.js

# Generate report
npm run test:report > axios-unit-tests-report.md

# Document any failures
if [ $? -ne 0 ]; then
  slack send "#staging-issues" "Axios 1.16.0 unit tests failed. Review: axios-unit-tests-report.md"
  exit 1
fi
```

**Day 4-5: Integration Testing**
```bash
# Run integration tests
npm run test -- tests/axios-1.16.0/integration-*.test.js
npm run test:report > axios-integration-tests-report.md

# Deploy to staging environment
docker build -t staging-canary:pr9 .
docker tag staging-canary:pr9 staging-registry.internal/canary:pr9-$(date +%s)
docker push staging-registry.internal/canary:pr9-$(date +%s)

# Deploy with canary strategy (5% traffic)
kubectl set image deployment/canary-green \
  canary=staging-registry.internal/canary:pr9-$(date +%s) \
  --namespace=staging

kubectl patch service canary -n staging -p \
  '{"spec":{"selector":{"canary":"5%"}}}'
```

#### Stage 2B: Staging Monitoring Week 1 (5% Traffic)

```bash
# Monitor for 3 days with 5% traffic
for day in {1..3}; do
  echo "=== Day $day of 5% traffic ==="
  
  # Check error rates
  curl -s "https://staging-monitoring.internal/api/metrics?query=error_rate" \
    | jq '.data'
  
  # Check response times
  curl -s "https://staging-monitoring.internal/api/metrics?query=p99_latency" \
    | jq '.data'
  
  # Check for Axios-related errors
  kubectl logs -l app=canary-green -n staging --tail=500 \
    | grep -i "axios\|http\|request" > daily-logs-day$day.log
  
  # Analyze form submissions (since form-data changed)
  grep -c "FormData\|multipart" daily-logs-day$day.log
  
  sleep 86400  # Wait 24 hours
done

# Expected metrics:
# - Error rate: <0.5% (same as baseline)
# - P99 latency: < baseline + 10%
# - No Axios parsing errors
# - FormData submissions succeed
```

#### Stage 2C: Gradual Traffic Increase

**After Day 3 of 5% monitoring:**
```bash
# Increase to 25% traffic (Day 4-5)
kubectl patch service canary -n staging -p \
  '{"spec":{"selector":{"canary":"25%"}}}'

./scripts/run-continuous-tests.sh --duration=48h

# Increase to 50% traffic (Day 6-7)
kubectl patch service canary -n staging -p \
  '{"spec":{"selector":{"canary":"50%"}}}'

./scripts/run-continuous-tests.sh --duration=48h

# Hold at 50% for week 2
```

#### Stage 2D: Week 2 - Full Load Testing

**Day 8-10: Load Tests**
```bash
# Run load tests with 100% traffic simulation
npm run test:load -- \
  --concurrency=1000 \
  --duration=1h \
  --profile=axios-intensive

# Expected results saved to load-test-results.json

# Analyze results
npm run analyze:load-test -- load-test-results.json

# Critical checks:
# ✓ No timeout errors from Axios
# ✓ Size limits respected under load
# ✓ Auth credentials properly handled
# ✓ Proxy routing stable
```

**Day 11-12: Form Data & Upload Tests**
```bash
# Test multipart form data thoroughly
npm run test:form-data -- --stress

# Test various file sizes
for size in 100KB 1MB 10MB 100MB; do
  echo "Testing $size upload..."
  ./scripts/upload-test.sh --size=$size --iterations=100
done

# Expected: All uploads succeed, errors tracked
```

**Day 13-14: Real Traffic Analysis**
```bash
# Pull metrics from week 2
./scripts/analyze-staging-week2.sh

# Generate comparison report
npm run report:comparison -- \
  --baseline=test-baseline.log \
  --current=week2-results.log

# Must show: No regression in error rates
```

#### Stage 2E: Final Approval (Day 14)

```bash
# Team sign-off checklist
cat > approval-checklist-pr9.md <<EOF
# Axios 1.16.0 Migration - Approval Checklist

## Week 1 Results (5%-50% traffic)
- [ ] Unit tests: All pass
- [ ] Integration tests: All pass
- [ ] Error rate < baseline: PASS/FAIL
- [ ] Performance metrics: OK/DEGRADED/IMPROVED
- [ ] No auth failures: YES/NO
- [ ] FormData uploads stable: YES/NO

## Week 2 Results (50%-100% simulation)
- [ ] Load test: PASS
- [ ] File upload tests: PASS
- [ ] Proxy scenarios: PASS
- [ ] Real traffic analysis: OK/CONCERNING

## Sign-Offs Required
- [ ] QA Lead
- [ ] API Team Lead
- [ ] DevOps Lead
- [ ] Security Team (if needed)

## Approved for Production?
- [ ] YES - Ready to merge
- [ ] NO - Require fixes (document below)

Notes: _______________
EOF

# Send for approval
slack send "#deployment-approvals" \
  "Axios 1.16.0 staging complete. Review: approval-checklist-pr9.md"
```

**Rollback if Issues Found:**
```bash
# Quick rollback (< 15 minutes)
kubectl rollout undo deployment/canary-green -n staging
kubectl patch service canary -n staging -p \
  '{"spec":{"selector":{"canary":"0%"}}}'

# Reset to previous version
npm ci --no-save  # Reset package-lock.json
npm run build
docker build -t staging-canary:stable .
kubectl set image deployment/canary-green \
  canary=staging-registry.internal/canary:stable \
  --namespace=staging

# Notify team
slack send "#incidents" \
  "@channel Rolled back Axios 1.16.0 due to issues. Investigating..."
```

---

### Phase 3: react-router PR #6 (Vite 8.0.13) - EXTENDED TRACK
**Duration:** 3-4 weeks | **Risk:** 🟡 MEDIUM | **Rollback Time:** 10 minutes

#### Stage 3A: Pre-Deployment Week 1 (Days 1-5)

**Day 1: Build Artifact Analysis**
```bash
# 1. Merge PR
git checkout -b staging/vite-pr6
git pull origin pull/6/head

# 2. Build with Vite 8.0.13
npm ci
npm run build 2>&1 | tee vite-build.log

# Analyze build output
./scripts/analyze-vite-build.sh vite-build.log

# Expected checks:
# - Build completes without errors
# - Build time documented (baseline)
# - Bundle size documented
# - Source maps generate correctly

# 3. Compare build artifacts
npm run analyze:bundle > bundle-report.json
npm run analyze:bundle -- --compare-baseline >> bundle-comparison.md

# Document changes
echo "# Vite 8.0.13 Build Analysis" > vite-analysis.md
echo "Build time: $(grep 'done in' vite-build.log)" >> vite-analysis.md
cat bundle-comparison.md >> vite-analysis.md
```

**Day 2-3: Worker & Module Tests**
```bash
# Test worker scripts specifically (Vite 8 changes this)
npm run test -- tests/vite-8.0/workers.test.js
npm run test -- tests/vite-8.0/esm-modules.test.js
npm run test -- tests/vite-8.0/dynamic-imports.test.js

# Generate report
npm run test:report > vite-module-tests-report.md

# Expected: All module loading scenarios work
```

**Day 4-5: Source Map & Debugging**
```bash
# Verify source maps work correctly
npm run build:dev

# Test source map quality
./scripts/test-sourcemaps.sh

# Expected output:
# ✓ Source maps generated
# ✓ Maps align with source files
# ✓ Debugging in browser works
# ✓ Stack traces point to correct lines

# Document any differences from Vite 6
npm run report:sourcemap-diff > sourcemap-diff.md
```

#### Stage 3B: Staging Deployment Week 1 (10% Traffic)

**Day 6-7: Initial Deployment**
```bash
# 1. Create Docker image
docker build -t staging-react-router:vite8 .
docker tag staging-react-router:vite8 \
  staging-registry.internal/react-router:vite8-$(date +%s)

# 2. Deploy to canary (10% traffic)
kubectl set image deployment/react-router-green \
  react-router=staging-registry.internal/react-router:vite8-$(date +%s) \
  --namespace=staging

kubectl rollout status deployment/react-router-green -n staging

# 3. Immediate health checks
./scripts/staging-health-check.sh react-router-green

# 4. Test critical routes
npm run test:smoke -- --environment=staging --filter=routes

# Expected: All critical routes respond
```

**Day 8-12: Monitoring at 10% Traffic**
```bash
# Continuous monitoring script
cat > scripts/vite8-monitor.sh <<'SCRIPT'
#!/bin/bash
DURATION=$((5 * 24 * 3600))  # 5 days
INTERVAL=300                  # 5 minutes

start_time=$(date +%s)

while true; do
  current_time=$(date +%s)
  elapsed=$((current_time - start_time))
  
  if [ $elapsed -gt $DURATION ]; then
    echo "Monitoring period complete"
    break
  fi
  
  # Check performance metrics
  echo "=== $(date) ==="
  
  # Build time variance (should be consistent)
  npm run build:measure | grep "done in"
  
  # Bundle size variance
  du -sh dist/
  
  # Runtime errors
  kubectl logs -l app=react-router-green -n staging \
    --since=5m | grep -i "error\|fail\|warn" || echo "No errors"
  
  # Browser console errors (from synthetic monitoring)
  curl -s "https://staging-monitoring.internal/api/browser-errors" \
    | jq '.data | length' || echo "Cannot fetch browser errors"
  
  sleep $INTERVAL
done
SCRIPT

chmod +x scripts/vite8-monitor.sh
./scripts/vite8-monitor.sh
```

#### Stage 3C: Week 2 - Increase to 50%

**Day 13-14: Scale Up**
```bash
# After 5 days at 10% with no issues, increase
kubectl patch service react-router -n staging -p \
  '{"spec":{"selector":{"version":"vite8-50%"}}}'

# Run extended load tests
npm run test:load -- \
  --concurrency=500 \
  --duration=2h \
  --profile=spa-with-workers

# Expected: Performance consistent with 10% traffic
```

**Day 15-19: Monitor at 50%**
```bash
# Continue monitoring
./scripts/vite8-monitor.sh

# Run synthetic user scenarios
npm run test:synthetic -- --duration=5d

# Check worker performance specifically
npm run test:workers -- --profile=heavy-load

# Verify dynamic imports work under load
npm run test:dynamic-imports -- --load-test
```

#### Stage 3D: Week 3 - Full Traffic & Stress Testing

**Day 20-21: 100% Traffic**
```bash
# Full rollover
kubectl patch service react-router -n staging -p \
  '{"spec":{"selector":{"version":"vite8-100%"}}}'

# Stress test scenarios
npm run test:stress -- --profile=peak-load

# Simulate real user load patterns
npm run test:scenarios -- \
  --scenario=peak-hour \
  --scenario=weekend \
  --scenario=holiday

# Expected: Response times < 1% degradation
```

**Day 22-28: Extended Stability**
```bash
# Run weekly suite
./scripts/weekly-vite8-validation.sh

# Weekly reports
npm run report:vite8 --week=1
npm run report:vite8 --week=2

# Generate final comparison
npm run report:comparison -- \
  --baseline=vite6-baseline.json \
  --current=vite8-week3.json \
  --output=vite8-final-report.md

# Must show: No regression in key metrics
```

#### Stage 3E: Final Approval (Day 28)

```bash
# Generate approval request
cat > approval-checklist-pr6.md <<EOF
# Vite 8.0.13 Migration - Approval Checklist

## Week 1 (10% traffic)
- [ ] Build succeeds consistently
- [ ] Source maps correct
- [ ] Workers function properly
- [ ] No browser console errors
- [ ] Performance baseline: PASS/FAIL

## Week 2 (50% traffic)
- [ ] Load tests: PASS
- [ ] Worker stress tests: PASS
- [ ] Dynamic imports stable: YES/NO
- [ ] Error rate unchanged: YES/NO

## Week 3 (100% traffic + stress)
- [ ] Peak load handling: OK
- [ ] Response times acceptable: YES/NO
- [ ] Memory usage stable: YES/NO
- [ ] No timeout issues: YES/NO

## Performance Metrics
- Build time variance: _____
- Bundle size change: _____
- P99 latency change: _____
- Error rate change: _____

## Final Sign-Offs
- [ ] Frontend Lead
- [ ] DevOps Lead
- [ ] QA Lead
- [ ] Performance Team (if needed)

## Approved for Production?
- [ ] YES - Ready for merge and production deployment
- [ ] NO - Require fixes (document below)

Concerns/Notes: _______________
EOF

slack send "#deployment-approvals" \
  "Vite 8.0.13 staging complete. Review: approval-checklist-pr6.md"
```

**Rollback if Issues Found:**
```bash
# Quick rollback (< 10 minutes)
kubectl rollout undo deployment/react-router-green -n staging

# Rebuild with Vite 6
git checkout package.json package-lock.json
npm ci
npm run build

docker build -t staging-react-router:vite6-stable .
kubectl set image deployment/react-router-green \
  react-router=staging-registry.internal/react-router:vite6-stable \
  --namespace=staging

# Clear build cache (important for Vite)
rm -rf node_modules/.vite
npm run build

slack send "#incidents" \
  "@channel Rolled back Vite 8.0.13 due to issues. Investigating..."
```

---

## 3. CROSS-DEPLOYMENT COORDINATION

### Deployment Sequence (All 3 PRs)

```
Week 1: ic-ui-kit PR #6 (PARALLEL)
├─ Day 1: Deploy to staging
├─ Day 1-2: Validation
└─ Day 2: Merge to main

Week 1-2: canary-ic-ui-kit PR #9 (SEQUENTIAL AFTER #6)
├─ Day 3: Start staging (5% traffic)
├─ Day 3-7: Ramp traffic (5% → 50%)
├─ Day 8-14: Load testing & approval
└─ Day 14: Ready for production (do not merge yet)

Week 2-4: react-router PR #6 (PARALLEL AFTER #9 APPROVAL)
├─ Day 15: Start staging (10% traffic)
├─ Day 15-20: Ramp traffic (10% → 100%)
├─ Day 21-28: Extended validation
└─ Day 28: Ready for production

Production Deployment (Week 5)
├─ Day 29: Merge PR #6 (ic-ui-kit)
├─ Day 30: Merge PR #9 (canary-ic-ui-kit) after #6 stable
└─ Day 31: Merge PR #6 (react-router) after #9 stable
```

### Parallel Monitoring Dashboard

```bash
# Create unified monitoring dashboard
./scripts/create-deployment-dashboard.sh

# Shows in real-time:
# - PR #6 (ic-ui-kit): Current status, errors, metrics
# - PR #9 (canary-ic-ui-kit): Traffic %, errors, latency
# - PR #6 (react-router): Traffic %, errors, latency
# - Dependency conflicts: Highlight if any arise
# - Overall system health: Capacity, errors, SLO compliance
```

---

## 4. COMMUNICATION PLAN

### Daily Standups
```bash
# 9:00 AM - Staging deployment status (async Slack)
slack send "#staging-deployments" \
  "Daily Standup: $(date +%Y-%m-%d)"
  "PR #6 (ic-ui-kit): [Status]"
  "PR #9 (canary-ic-ui-kit): [Status]"
  "PR #6 (react-router): [Status]"
  "Blockers: [List]"

# 4:00 PM - End of day summary
slack send "#staging-deployments" \
  "EOD Summary: Metrics dashboard: [link]"
```

### Escalation Path
```
Issue Detected
    ↓
Severity: CRITICAL (system down) → On-Call + Lead + VP
Severity: HIGH (partial outage) → Lead + QA Manager
Severity: MEDIUM (degradation) → QA Lead + Engineer
Severity: LOW (minor issue) → Document in issue tracker
```

---

## 5. SIGN-OFF CHECKLIST

### Before Staging Deployment
- [ ] All code reviews completed
- [ ] CI/CD pipeline green
- [ ] Security scans passed
- [ ] Backup created
- [ ] Monitoring configured
- [ ] Runbooks updated
- [ ] Team notified of schedule
- [ ] Stakeholders briefed

### Before Production Deployment
- [ ] Staging validation complete
- [ ] Approval checklist signed
- [ ] Rollback procedure tested
- [ ] Communication templates ready
- [ ] On-call engineer assigned
- [ ] Maintenance window scheduled (if needed)
- [ ] Customers notified (if needed)
- [ ] DNS/CDN changes prepared (if needed)

---

**Document Version:** 1.0
**Last Updated:** 2026-07-06
**Schedule:** Deployments begin 2026-07-13
