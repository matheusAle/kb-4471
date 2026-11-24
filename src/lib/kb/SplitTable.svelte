<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // Engineering update, 5 Oct: `fix(dates): parse dd/mm for eu locales` shipped; the booking-v3 form was retired.
  // Day-first locales outside the EU are not covered by that fix.
  const OUTSIDE_EU = ['en-GB'];
  const DESK: Record<string, string> = { parser: 'fixed', parserOutsideEu: 'open', hint: "won't fix", unplaced: '' };

  let { ticket }: { ticket: number } = $props();

  // Two problems: the desk's own cause rule. The desk reads some reports as a one-day move (desk-notes),
  // but they are filed under the cause their customer has, so they are footnoted, not split out.
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report) => {
    const cause = causeOf(r, tenantOf(r)) ?? 'unplaced';
    return cause === 'parser' && OUTSIDE_EU.includes(tenantOf(r)?.locale ?? '') ? 'parserOutsideEu' : cause;
  };
  const rows = $derived(
    ['parser', 'parserOutsideEu', 'hint', 'unplaced']
      .map((p) => {
        const rs = merged.filter((r) => problemOf(r) === p);
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem: p,
          n: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          first: days[0],
          last: days[days.length - 1],
          taggedRight: rs.filter((r) => r.meta.intakeTag === (p === 'parserOutsideEu' ? 'parser' : p)).length,
          oneDay: rs.filter((r) => notes.oneDayMove.includes(r.id)).length,
        };
      })
      .filter((x) => x.n > 0),
  );
</script>

<p><strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>customers</th><th>seen</th><th>intake tagged it right</th><th>desk</th></tr></thead>
  <tbody>
    {#each rows as x (x.problem)}
      <tr><td><code>{x.problem}</code></td><td>{x.n}</td><td>{x.tenants}</td><td>{x.first} → {x.last}</td><td>{x.taggedRight} of {x.n}</td><td>{DESK[x.problem]}</td></tr>
    {/each}
  </tbody>
</table>

{#each rows as x (x.problem)}
  {#if x.oneDay}
    <p class="note">{x.oneDay} of the {x.n} <code>{x.problem}</code> reports read as a one-day move. They are filed under <code>{x.problem}</code> by customer.</p>
  {/if}
{/each}
