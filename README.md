# Apache Flink Kubernetes Operator

A Kubernetes operator for Apache Flink, implemented in Java. It allows users to manage Flink applications and their lifecycle through native k8s tooling like kubectl.

<img alt="Operator Overview" width="100%" src="docs/static/img/overview.svg">

## Documentation & Getting Started

Please check out the full [documentation](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/), hosted by the
[ASF](https://www.apache.org/), for detailed information and user guides.

Check our [quick-start](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/) guide for simple setup instructions to get you started with the operator.

## Features at a glance

 - Deploy and monitor Flink Application, Session and Job deployments
 - Upgrade, suspend and delete deployments
 - Full logging and metrics integration
 - Flexible deployments and native integration with Kubernetes tooling
 - Flink Job Autoscaler

For the complete feature-set please refer to our [documentation](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/concepts/overview/).

## About cron batch-job
provide a sample idea on FlinkDeploymentController
- add 'job.execute.cron' annotation on FlinkDeployment CR
- add ThreadPoolTaskScheduler on FlinkDeploymentController
- if 'job.execute.cron' annotation exists on FlinkDeployment reconciling, submit a schedule job
- cron job triggered by scheduler named as {appname}-{MMddHHmm}, and triggerred jobs are records on k8s event records 

```
    private final Map<String, Date> cronRecords = new HashMap<>();
    private final ThreadPoolTaskScheduler cronScheduler = new ThreadPoolTaskScheduler();
    
    @Override
    public UpdateControl<FlinkDeployment> reconcile(FlinkDeployment flinkApp, Context josdkContext)
            throws Exception {
        if (canaryResourceManager.handleCanaryResourceReconciliation(flinkApp)) {
            return UpdateControl.noUpdate();
        }
        String cron = flinkApp.getMetadata().getAnnotations().get("job.execute.cron");
        if (cron != null && !cron.isEmpty()) {
            try {
                this.reconcileCronJob(flinkApp, josdkContext, cron);
            } catch (Exception e) {
                LOG.error("failed to execute cronjob", e);
            }
            this.cronScheduler.schedule(() -> {
                try {
                    this.reconcileCronJob(flinkApp, josdkContext, cron);
                } catch (Exception e) {
                    LOG.error("failed to reconcile cronjob", e);
                }
            }, new CronTrigger(cron));
            return UpdateControl.noUpdate();
        } else {
            return reconcileApp(flinkApp, josdkContext);
        }
    }
    
    private void reconcileCronJob(FlinkDeployment flinkApp, Context josdkContext, String cron) throws Exception {
        String applicationName = flinkApp.getMetadata().getName();
        CronExpression cronExpr = new CronExpression(cron);
        Date triggerTime = cronExpr.getPrevFireTime(new Date());
        Date lastTriggerTime = cronRecords.get(applicationName);
        if (lastTriggerTime == null) {
            LOG.info("application[{}] first trigger, trigger time {}", applicationName, triggerTime);
        } else {
            String staleAppName = String.format("%s-%s", applicationName, DateFormatUtils.format(lastTriggerTime, "MMddHHmm"));
            FlinkDeployment staleCopy = ReconciliationUtils.clone(flinkApp);
            staleCopy.getMetadata().setName(staleAppName);
            LOG.info("start to cleanup stale job[{}]", staleAppName);
            try {
                this.cleanup(staleCopy, josdkContext);
                eventRecorder.triggerEvent(
                        flinkApp,
                        EventRecorder.Type.Normal,
                        EventRecorder.Reason.Cleanup,
                        EventRecorder.Component.Operator,
                        "Clean up " + staleAppName);
            } catch (Exception e){
                LOG.error("failed to cleanup stale job[{}], err: {}", staleAppName, e);
            }
        }

        FlinkDeployment previousDeployment = ReconciliationUtils.clone(flinkApp);
        FlinkDeployment copy = ReconciliationUtils.clone(flinkApp);
        String newAppName = String.format("%s-%s", applicationName, DateFormatUtils.format(triggerTime, "MMddHHmm"));
        copy.getMetadata().setName(newAppName);
        copy.getStatus().getJobStatus().setState(JobStatus.CREATED.name());
        FlinkResourceContext<FlinkDeployment> ctx = ctxFactory.getResourceContext(copy, josdkContext);
        reconcilerFactory.getOrCreate(copy).reconcile(ctx);

        cronRecords.put(applicationName, triggerTime);
        eventRecorder.triggerEvent(
                flinkApp,
                EventRecorder.Type.Normal,
                EventRecorder.Reason.JobStatusChanged,
                EventRecorder.Component.Operator,
                "Submit new job " + newAppName);

        flinkApp.getStatus().getJobStatus().setState("CRON_SCHEDULED");
        statusRecorder.patchAndCacheStatus(flinkApp);
        ReconciliationUtils.toUpdateControl(
                ctx.getOperatorConfig(), flinkApp, previousDeployment, true);
    }

    private UpdateControl<FlinkDeployment> reconcileApp(FlinkDeployment flinkApp, Context josdkContext) {
        LOG.debug("Starting reconciliation");
        FlinkResourceContext<FlinkDeployment> ctx = ctxFactory.getResourceContext(flinkApp, josdkContext);
        statusRecorder.updateStatusFromCache(flinkApp);
        FlinkDeployment previousDeployment = ReconciliationUtils.clone(flinkApp);
        try {
            observerFactory.getOrCreate(flinkApp).observe(ctx);
            if (!validateDeployment(ctx)) {
                statusRecorder.patchAndCacheStatus(flinkApp);
                return ReconciliationUtils.toUpdateControl(
                        ctx.getOperatorConfig(), flinkApp, previousDeployment, false);
            }
            statusRecorder.patchAndCacheStatus(flinkApp);
            reconcilerFactory.getOrCreate(flinkApp).reconcile(ctx);
        } catch (RecoveryFailureException rfe) {
            handleRecoveryFailed(ctx, rfe);
        } catch (DeploymentFailedException dfe) {
            handleDeploymentFailed(ctx, dfe);
        } catch (Exception e) {
            eventRecorder.triggerEvent(
                    flinkApp,
                    EventRecorder.Type.Warning,
                    "ClusterDeploymentException",
                    e.getMessage(),
                    EventRecorder.Component.JobManagerDeployment);
            throw new ReconciliationException(e);
        }

        LOG.debug("End of reconciliation");
        statusRecorder.patchAndCacheStatus(flinkApp);
        return ReconciliationUtils.toUpdateControl(
                ctx.getOperatorConfig(), flinkApp, previousDeployment, true);
    }
```


## Project Status

### Project status: Production Ready

### Current API version: `v1beta1`

To download the latest stable version please visit the [Flink Downloads Page](https://flink.apache.org/downloads.html).
The official operator images are also available on [Dockerhub](https://hub.docker.com/r/apache/flink-kubernetes-operator/tags).

Please check out our docs to read about the [upgrade process](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/operations/upgrade/) and our [backward compatibility guarantees](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/operations/compatibility/).

## Support

Don’t hesitate to ask!

Contact the developers and community on the [mailing lists](https://flink.apache.org/community.html#mailing-lists) if you need any help.

[Open an issue](https://issues.apache.org/jira/browse/FLINK) if you found a bug in Flink.

## Contributing

You can learn more about how to contribute in the [Apache Flink website](https://flink.apache.org/contributing/how-to-contribute.html). For code contributions, please read carefully the [Contributing Code](https://flink.apache.org/contributing/contribute-code.html) section for an overview of ongoing community work.

## License

The code in this repository is licensed under the [Apache Software License 2](LICENSE).
