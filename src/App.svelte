<script>
	const { read, utils } = globalThis.XLSX;

	function upload(e, callback) {
		const file = e.currentTarget.files[0];
		file.arrayBuffer().then((rawTrans) => {
			const fileTrans = read(rawTrans, { cellDates: true });
			const sheetName = fileTrans.SheetNames[0];
			const worksheet = fileTrans.Sheets[sheetName];
			callback(utils.sheet_to_json(worksheet, { header: 1 }))
		});
	}

	let dataGL = $state([[]]);

	let report = $derived.by(() => {
		let desc = {}
		let pay = []
		let receive = []
		let rowindex = 0
		for (const cells of dataGL.slice(1)) {
			if (['K0','KA','KC','KL'].includes(cells[5])) {
				desc[cells[3]] = cells[15]
				receive.push({ date: cells[6], group: cells[12], desc: cells[15] })
			}
			if (cells[5] == 'PM') {
				pay.push({ date: cells[6], group: cells[12], desc: desc[cells[11].slice(4)] })
			}
			rowindex += 1
		}
		return { desc, pay, receive }
	});
</script>

<div class="">
	<label for="">
		NFI_DISPLAY_L การแสดงบรรทัดรายการบัญชีแยกประเภททั่วไป
		<input
			type="file"
			class=""
			accept="xlsx"
			onchange={(e) => {
				upload(e, (result) => {
					dataGL = result.slice(6)
				});
			}}
		/>
	</label>
</div>

<div class="">
	<table>
		<tbody>
			{#each report.pay as obj}
				<tr>
					<td>{obj.date}</td>
					<td>{obj.group}</td>
					<td>{obj.desc}</td>
				</tr>
			{/each}
		</tbody>
	</table>
</div>