---
link: https://arxiv.org/pdf/2304.07177.pdf
---

# TLDR;

This research benchmarks Google Cloud Functions over two months, finding significant performance variations throughout the day, particularly in request-response latency and unexpected cold starts. The paper proposes a methodology for evaluating these temporal performance variations in cloud FaaS platforms and highlights implications for practitioners running applications on such platforms. The research artifacts are available as open-source for replication and extension.

---

Due to the high level of abstraction, developers rely on the cloud platform to offer a consistent service level, as decreased performance leads to higher latency and higher cost given the pay-per-use model.

Our results show that diurnal patterns can lead to performance differences of up to 15%, and that the frequency of unexpected cold starts increases threefold during the start of the week.

Somewhat counterintuitively, FaaS users must actually pay more when a FaaS platform underperforms and latency is higher, as billed function execution is also longer.

Billed Duration of warm calls to float with 128MB memory. During working hours, the billed duration increases by up to 15%.

Relative frequency of unexpected cold starts, averaged by the hour of the week they happened in. On Mondays during the day, up to 13% of invocations can be unexpected cold starts, compared to less than 5% during the night and on weekends.
