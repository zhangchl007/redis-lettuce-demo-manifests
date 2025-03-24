# GitOps Demo
redis-lettuce-demo-manifests

# validate application deployment
```bash
curl -X POST -d @- -H 'Content-Type: application/json' \
http://localhost:8080/api/v1/otp/generate <<'EOF'
{
	"email": "user2@gmail.com"
}
EOF
```

