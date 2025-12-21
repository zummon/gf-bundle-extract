<script>
	const { read, utils } = globalThis.XLSX;
	const routes = [
		{ value: "เงินในงบฯ", callback() {} },
		{ value: "อื่นๆ", callback() {} },
	];
	const budgetTypes = { 0: "กลาง", 1: "สรก.", 6: "เงินประกัน" };

	function eoMonth(startDate, months) {
		const d = new Date(startDate);
		return new Date(d.getFullYear(), d.getMonth() + months + 1, 0);
	}
	function formatMoney(value, option) {
		value = Number(value);
		return value == 0
			? ""
			: value.toLocaleString("th", {
					minimumFractionDigits: 2,
					maximumFractionDigits: 2,
					...option,
				});
	}
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
	function restructJournal(aoa) {
		let journal = [];
		for (const cells of aoa) {
			const [, account] = cells[7].split("\n");

			if (!account) {
				const [, accountName, desc] = cells[1].split("\n");
				const [docNo] = cells[2].split("\n");
				const [docDate, accountCode, sourceFund] = cells[4].split("\n");
				const [docType] = cells[5].split("\n");
				const [refNo] = cells[6].split("\n");
				const [, debit] = cells[11].split("\n");
				const [, credit] = cells[12].split("\n");
				const [day, month, year] = docDate.split(".");
				journal.push([
					,
					accountName,
					desc,
					docNo,
					new Date(year - 543, month - 1, day),
					accountCode,
					sourceFund,
					docType,
					refNo,
					account,
					,
					Number(debit.replace(/[^0-9.-]+/g, "")),
					,
					Number(credit.replace(/[^0-9.-]+/g, "")),
				]);
			}
		}
		return journal;
	}

	let journal = $state([]);
	let route = $state(routes[0].value);
	let pickDocTypes = $state([]);
	let docTypes = $state([]);

	let allowed = $derived.by(() => {
		let detail = {};
		let corts = {};
		let byDate = {};
		let rowindex = 0;
		for (const cells of journal.slice(1)) {
			const [
				,
				accountName,
				desc,
				docNo,
				docDate,
				accountCode,
				sourceFund,
				docType,
				refNo,
				account,
				,
				debit,
				,
				credit,
			] = cells;
			if (!account) {
				const isReceiving = ["K0", "KA", "KC", "KL", "KZ"].includes(docType);
				const isPaying = ["PM"].includes(docType);
				const cort = refNo.slice(-3) + "/" + refNo.slice(1, 3);
				const referral = refNo.slice(-10);
				if (isReceiving) {
					detail[docNo] = desc;
					corts[docNo] = cort;
				}
				if (isPaying || isReceiving) {
					if (!byDate[docDate]) {
						byDate[docDate] = [];
					}
					byDate[docDate].push({
						cort: isNaN(refNo) ? cort : corts[referral],
						budgetType: budgetTypes[sourceFund.slice(3, 4)],
						desc: isReceiving ? desc : detail[referral],
						debit,
						credit,
						docNo: isPaying ? referral : docNo,
						payRef: isPaying ? docNo : "",
						accountName,
						accountCode,
					});
				}
			}

			rowindex += 1;
		}
		return { byDate };
	});
</script>

<div class="p-4 flex flex-wrap gap-4 print:hidden select-none">
	<div class="">
		<label>
			NGL_RPT001 รายงานสมุดรายวันทั่วไป (รายเดือน)
			<input
				type="file"
				class="cursor-pointer text-cyan-500"
				accept="xlsx"
				onchange={(e) => {
					upload(e, (aoa) => {
						clear();
						journal = restructJournal(aoa);
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
	{#each routes as { value, callback }}
		<div class="">
			<button
				class="cursor-pointer {route == value
					? 'bg-zinc-500'
					: 'bg-cyan-500'} font-semibold text-white rounded px-1"
				onclick={() => {
					route = value;
					callback();
				}}>{value}</button
			>
		</div>
	{/each}
	<div class="">
		<button
			class="cursor-pointer bg-violet-500 font-semibold text-white rounded px-1"
			onclick={() => {
				print();
			}}>Print</button
		>
	</div>
</div>

<div class="px-4 flex flex-wrap gap-4 print:hidden select-none">
	<div class=""></div>
</div>

<div class="p-4 print:p-0 text-sm">
	{#if route == routes[0].value}
		{@const firstDate = Object.keys(allowed.byDate)[0]}
		<table class="overflow-auto w-full">
			<thead class="text-center">
				<tr>
					<td class="" colspan="8">
						บัญชีธนาคาร เงินงบประมาณ ประจำเดือน {formatDate(
							eoMonth(firstDate, 0),
							{ day: undefined, month: "long" },
						)} ปีงบประมาณ {eoMonth(firstDate, 3).getFullYear() + 543}
					</td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
				</tr>
				<tr>
					<td class="border">วันที่</td>
					<td class="border">เลขที่ฎีกา</td>
					<td class="border">ประเภทจ่าย</td>
					<td class="border">งบ</td>
					<td class="border">รายการ</td>
					<td class="border">เดบิต</td>
					<td class="border">เครดิต</td>
					<td class="border">ยอดคงเหลือ</td>
					<td class="print:hidden">วัน เดือน ปี</td>
					<td class="print:hidden">รหัสบัญชี</td>
					<td class="print:hidden">ชื่อบัญชี</td>
					<td class="print:hidden">เลขที่เอกสาร GF</td>
					<td class="print:hidden">เลขที่เอกสารจ่าย GF</td>
				</tr>
			</thead>
			<tbody>
				<tr class="">
					<td class="border-x text-center" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-center" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-center" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-center" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-center" style="border-bottom: 1px dotted;">
						ยอดยกมา
					</td>
					<td class="border-x text-right" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-right" style="border-bottom: 1px dotted;"
					></td>
					<td class="border-x text-right" style="border-bottom: 1px dotted;">
						{formatMoney(0)}
					</td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
				</tr>
				{#each Object.entries(allowed.byDate) as [date, arr]}
					{@const formatedDate = formatDate(date, { year: undefined })}
					{@const { debit, credit } = arr.reduce(
						(prev, curr) => {
							prev.debit += curr.debit;
							prev.credit += curr.credit;
							return prev;
						},
						{ debit: 0, credit: 0 },
					)}
					{#each arr as obj, index}
						<tr class="">
							<td
								class="border-x text-center text-nowrap border-black"
								style="border-bottom: 1px dotted var(--color-black);"
							>
								{index == 0 ? formatedDate : ""}
							</td>
							<td
								class="border-x text-center"
								style="border-bottom: 1px dotted;"
							>
								{obj.cort}
							</td>
							<td
								class="border-x text-center"
								style="border-bottom: 1px dotted;"
							></td>
							<td
								class="border-x text-center"
								style="border-bottom: 1px dotted;"
							>
								{obj.budgetType}
							</td>
							<td class="border-x" style="border-bottom: 1px dotted;">
								{obj.desc}
							</td>
							<td
								class="border-x text-right"
								style="border-bottom: 1px dotted;"
							>
								{formatMoney(obj.debit)}
							</td>
							<td
								class="border-x text-right"
								style="border-bottom: 1px dotted;"
							>
								{formatMoney(obj.credit)}
							</td>
							<td class="border-x text-right" style="border-bottom: 1px dotted;"
							></td>
							<td class="print:hidden text-nowrap">
								{formatDate(date, { calendar: "gregory" })}
							</td>
							<td class="print:hidden">{obj.accountCode}</td>
							<td class="print:hidden">{obj.accountName}</td>
							<td class="print:hidden">{obj.docNo}</td>
							<td class="print:hidden">{obj.payRef}</td>
						</tr>
					{/each}
					<tr class="">
						<td class="border">{formatedDate}</td>
						<td class="border"></td>
						<td class="border"></td>
						<td class="border"></td>
						<td class="border"></td>
						<td class="border text-right">{formatMoney(debit)}</td>
						<td class="border text-right">{formatMoney(credit)}</td>
						<td class="border"></td>
						<td class="print:hidden"></td>
						<td class="print:hidden"></td>
						<td class="print:hidden"></td>
						<td class="print:hidden"></td>
						<td class="print:hidden"></td>
					</tr>
				{/each}
				<tr class="">
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border text-center">รวมทั้งเดือน</td>
					<td class="border text-right">{formatMoney(0)}</td>
					<td class="border text-right">{formatMoney(0)}</td>
					<td class="border"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
				</tr>
				<tr class="">
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border"></td>
					<td class="border text-center">รวมตั้งแต่ต้นปี</td>
					<td class="border text-right">{formatMoney(0)}</td>
					<td class="border text-right">{formatMoney(0)}</td>
					<td class="border"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
					<td class="print:hidden"></td>
				</tr>
			</tbody>
		</table>
	{/if}
</div>
