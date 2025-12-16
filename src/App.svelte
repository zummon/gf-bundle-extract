<script>
	const { read, utils } = globalThis.XLSX;
	const routes = ["เงินในงบฯ"];
	const budgetTypes = { 0: "กลาง", 1: "สรก.", 6: "เงินประกัน" };

	function formatDate(value, option) {
		return new Date(value).toLocaleDateString("th", {
			day: "numeric",
			month: "short",
			year: "numeric",
			...option,
		});
	}
	function upload(e, callback) {
		const file = e.currentTarget.files[0];
		file.arrayBuffer().then((rawTrans) => {
			const fileTrans = read(rawTrans, { cellDates: true });
			const sheetName = fileTrans.SheetNames[0];
			const worksheet = fileTrans.Sheets[sheetName];
			let aoa = utils.sheet_to_json(worksheet, { header: 1 });
			callback(aoa);
		});
	}
	function clear() {
		journal = [];
	}

	let journal = $state([]);
	let route = $state(routes[0]);

	let report = $derived.by(() => {
		let detail = {};
		let corts = {};
		let allowed = [];
		let rowindex = 0;
		for (const cells of journal.slice(1)) {
			let [docDate, accountCode, sourceFund] = cells[4].split("\n");
			if (!isNaN(accountCode)) {
				const [, accountName, desc] = cells[1].split("\n");
				const [docNo] = cells[2].split("\n");
				const [docType] = cells[5].split("\n");
				const [refNo] = cells[6].split("\n");
				const [, account] = cells[7].split("\n");
				const [, debit] = cells[11].split("\n");
				const [, credit] = cells[12].split("\n");
				const [day, indexmonth, year] = docDate.split(".");
				docDate = new Date(year - 543, indexmonth, day);
				const isReceiving = ["K0", "KA", "KC", "KL"].includes(docType);
				const isPaying = ["PM"].includes(docType);
				const cort = refNo.slice(-3) + "/" + refNo.slice(1, 3);
				const referral = refNo.slice(-10);
				if (isReceiving) {
					detail[docNo] = desc;
					corts[docNo] = cort;
				}
				if (isPaying || isReceiving) {
					allowed.push({
						docDate,
						docNo: isPaying ? referral : docNo,
						cort: isNaN(refNo) ? cort : corts[referral],
						budgetType: budgetTypes[sourceFund.slice(3, 4)],
						desc: isReceiving ? desc : detail[referral],
						debit,
						credit,
					});
				}
			}

			rowindex += 1;
		}
		return { allowed };
	});
</script>

<div class="p-4 flex flex-wrap gap-4 print:hidden">
	<div class="">
		<label>
			NGL_RPT001 รายงานสมุดรายวันทั่วไป
			<input
				type="file"
				class="cursor-pointer"
				accept="xlsx"
				onchange={(e) => {
					upload(e, (aoa) => {
						clear()
						journal = [
							aoa[0],
							...aoa.slice(1).sort((a,b) => {
								[, , a] = a[1].split("\n");
								[, , b] = b[1].split("\n");
								return b - a
							})
						]
					});
				}}
			/>
		</label>
	</div>
	<div class="">
		<button
			class="cursor-pointer {journal[0]
				? 'bg-orange-500'
				: 'bg-zinc-500'} font-semibold text-white rounded px-1"
			onclick={() => {
				clear();
			}}>Clear</button
		>
	</div>
	{#each routes as value}
		<div class="">
			<button
				class="cursor-pointer {route == value
					? 'bg-zinc-500'
					: 'bg-cyan-500'} font-semibold text-white rounded px-1"
				onclick={() => {
					route = value;
				}}>{value}</button
			>
		</div>
	{/each}
	<div class="">
		<button
			class="cursor-pointer bg-cyan-500 font-semibold text-white rounded px-1"
			onclick={() => {
				print();
			}}>Print</button
		>
	</div>
</div>

<div class="px-4 flex flex-wrap gap-4 print:hidden">
	<div class="">...</div>
</div>

<div class="p-4 print:p-0">
	{#if route == routes[0]}
		<table class="overflow-auto w-full">
			<thead class="text-center">
				<tr>
					<td class="border">วันที่</td>
					<td class="border">เลขที่ฎีกา</td>
					<!-- <td class="border">ประเภทจ่าย</td> -->
					<td class="border">งบ</td>
					<td class="border">รายการ</td>
					<td class="border">เดบิต</td>
					<td class="border">เครดิต</td>
					<!-- <td class="border">ยอดคงเหลือ</td> -->
					<td class="print:hidden">เลขที่เอกสาร GF</td>
				</tr>
			</thead>
			<tbody>
				{#each report.allowed as obj}
					<tr class="">
						<td class="border-x text-nowrap" style="border-bottom: 1px dotted;"
							>{formatDate(obj.docDate, { year: undefined })}</td
						>
						<td class="border-x" style="border-bottom: 1px dotted;"
							>{obj.cort}</td
						>
						<!-- <td class="border-x" style="border-bottom: 1px dotted;"></td> -->
						<td class="border-x" style="border-bottom: 1px dotted;">
							{obj.budgetType}
						</td>
						<td class="border-x" style="border-bottom: 1px dotted;"
							>{obj.desc}</td
						>
						<td class="border-x text-right" style="border-bottom: 1px dotted;"
							>{obj.debit}</td
						>
						<td class="border-x text-right" style="border-bottom: 1px dotted;"
							>{obj.credit}</td
						>
						<!-- <td class="border-x" style="border-bottom: 1px dotted;"></td> -->
						<td class="print:hidden">{obj.docNo}</td>
					</tr>
				{/each}
			</tbody>
			<tfoot>
				<tr class="">
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="print:hidden"></td>
				</tr>
			</tfoot>
		</table>
	{/if}
</div>
