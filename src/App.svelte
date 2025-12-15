<script>
  
</script>

<div class="">
	<input
	type="file"
	class="file-input"
	accept="xlsx"
	onchange={(e) => {
		const file = e.currentTarget.files[0];
		file.arrayBuffer().then((rawTrans) => {
			const fileTrans = read(rawTrans, { cellDates: true });
			const sheetName = fileTrans.SheetNames[0];
			const worksheet = fileTrans.Sheets[sheetName];
			utils.sheet_to_json(worksheet);
		});
	}}
/>
</div>