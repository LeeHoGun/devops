{
	echo "## curl 확인 결과"
	echo '```'
	for p in 8091 8092 8093; do
		echo "$ curl http://localhost:$p"
		curl -s http://localhost:$p
	done
	echo '```'
} >> week05/README.md
