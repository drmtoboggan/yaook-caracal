https://gitlab.com/yaook/operator 


## Bypass insecure flag
cat >> ~/admin-openrc <<'EOF'

# Always use --insecure flag
alias openstack='command openstack --insecure'
EOF

source ~/admin-openrc
openstack image list
