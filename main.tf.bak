terraform {
    required_providers {
      azurerm = {
          source = "hashicorp/azurerm"
          version = ">= 2.26"
      }
    }
    required_version = ">= 0.14.9"
}

provider "azurerm" {
  features {}
}

variable "web_frontend_count" {
    type = string
    description = "Number of web frontends to create"
    default = "2"
}

variable "db_backend_count" {
    type = string
    description = "Number of db backends to create"
    default = "1"
}

variable "admin_username" {
    type = string
    description = "Admin User Name for VM"
    default = "daveops"  
}

variable "admin_pubkey" {
    type = string
    description = "Admin User pubkey for VM"
    default = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9HuFS3LIsxmAs+njC8rNCXhmIC12y/y9Ebq3fiIxGDUcC765Km6og5aH1kRAwhnxdPxALcKvT1EkHQjsrCqNvkNwfvEs9VTLv4Qgf5kyRgzYixP4f8kfHhTZ2icCTateI3S2rgwrFmKs2DULUp3NGIfXytzyNqmW/YNurPi5tUqX7Tv3o8Fz8kPy8TVHT7vqdLQdHJp0X2tDkOjntX7FB5/te+/y9E71/Mav0ef4kqSDXnVQbHhZVRJxqR7jY76pqhkxOGAuHVDE/fWE76Dp95S8r9boVGFzCir1kNcpV6S0bTdhX7/wqS5/9Q6vY5rFrqNPvvcmqfvSXszR+SRmj dave@ubuntu-server"  
}

resource "azurerm_resource_group" "rg" {
  name = "ahkTFResourceGroup"
  location = "westus2"

  tags = {
    Environment = "Dave's Deploy Presentation"
    Team = "DaveOps"
  }
}

resource "azurerm_virtual_network" "vnet" {
  name = "DaveTFVnet"
  address_space = ["10.0.0.0/16"]
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "web-subnet" {
    name = "DaveTF-Web-Subnet"
    resource_group_name = azurerm_resource_group.rg.name
    virtual_network_name = azurerm_virtual_network.vnet.name
    address_prefixes = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "db-subnet" {
    name = "DaveTF-Db-Subnet"
    resource_group_name = azurerm_resource_group.rg.name
    virtual_network_name = azurerm_virtual_network.vnet.name
    address_prefixes = ["10.0.2.0/24"]
}

resource "azurerm_public_ip" "public-web-ip" {
    name = "daveTFPublic-web-IP-${count.index}"
    location = azurerm_resource_group.rg.location
    allocation_method = "Dynamic"
    resource_group_name = azurerm_resource_group.rg.name
    count = var.web_frontend_count
}

resource "azurerm_public_ip" "public-db-ip" {
    name = "daveTFPublic-db-IP-${count.index}"
    location = azurerm_resource_group.rg.location
    allocation_method = "Dynamic"
    resource_group_name = azurerm_resource_group.rg.name
    count = var.db_backend_count
}

resource "azurerm_network_security_group" "nsg" {
    name = "DaveTFNSG"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_network_security_rule" "ssh-rule" {
    name = "SSHIn"
    priority = 1001
    direction = "Inbound"
    access = "Allow"
    protocol = "Tcp"
    source_port_range = "*"
    destination_port_range = "22"
    source_address_prefix = "*"
    destination_address_prefix = "*"
    resource_group_name = azurerm_resource_group.rg.name
    network_security_group_name = azurerm_network_security_group.nsg.name
}

//For testing direct connection

/*
resource "azurerm_network_security_rule" "web-rule" {
    name = "WebIn"
    priority = 1002
    direction = "Inbound"
    access = "Allow"
    protocol = "Tcp"
    source_port_range = "*"
    destination_port_ranges = ["80","443"]
    source_address_prefix = "*"
    destination_address_prefix = "10.0.1.0/24"
    resource_group_name = azurerm_resource_group.rg.name
    network_security_group_name = azurerm_network_security_group.nsg.name
}
*/

resource "azurerm_network_interface" "web-nic" {
    name = "DaveTF-Web-NIC-${count.index}"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    count = var.web_frontend_count

    ip_configuration {
        name = "DaveTF-Web-NICConfig"
        subnet_id = azurerm_subnet.web-subnet.id
        private_ip_address_allocation = "dynamic"
        public_ip_address_id = azurerm_public_ip.public-web-ip[count.index].id
    }
}

resource "azurerm_network_interface" "db-nic" {
    name = "DaveTF-Db-NIC-${count.index}"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    count = var.db_backend_count

    ip_configuration {
        name = "DaveTF-Web-NICConfig"
        subnet_id = azurerm_subnet.web-subnet.id
        private_ip_address_allocation = "dynamic"
        public_ip_address_id = azurerm_public_ip.public-db-ip[count.index].id
    }
}

resource "azurerm_virtual_machine" "web-vm" {
    name = "DaveTF-Web-VM-${count.index}"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    network_interface_ids = [azurerm_network_interface.web-nic[count.index].id]
    vm_size = "Standard_B1ls"
    count = var.web_frontend_count
    availability_set_id = azurerm_availability_set.web-set.id
    
    storage_os_disk {
        name = "DaveTF-web-OSDisk-${count.index}"
        caching = "ReadWrite"
        create_option = "FromImage"
        managed_disk_type = "Standard_LRS"
    }

    storage_image_reference {
        publisher = "Canonical"
        offer = "UbuntuServer"
        sku = "18.04-LTS"
        version = "latest"
    }

    os_profile {
        computer_name = "DaveTF-Web-VM-${count.index}"
        admin_username = var.admin_username        
    }

    os_profile_linux_config {
        disable_password_authentication = true
        ssh_keys {
            path = "/home/${var.admin_username}/.ssh/authorized_keys"
            key_data = var.admin_pubkey
        }
    }
}


resource "azurerm_virtual_machine" "db-vm" {
    name = "DaveTF-db-VM-${count.index}"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    network_interface_ids = [azurerm_network_interface.db-nic[count.index].id]
    vm_size = "Standard_B1ls"
    count = var.db_backend_count

    storage_os_disk {
        name = "DaveTF-db-OSDisk-${count.index}"
        caching = "ReadWrite"
        create_option = "FromImage"
        managed_disk_type = "Standard_LRS"
    }

    storage_image_reference {
        publisher = "Canonical"
        offer = "UbuntuServer"
        sku = "18.04-LTS"
        version = "latest"
    }

    os_profile {
        computer_name = "DaveTF-Db-VM-${count.index}"
        admin_username = var.admin_username        
    }

    os_profile_linux_config {
        disable_password_authentication = true
        ssh_keys {
            path = "/home/${var.admin_username}/.ssh/authorized_keys"
            key_data = var.admin_pubkey
        }
    }
}

data "azurerm_public_ip" "web_ip" {
    name = azurerm_public_ip.public-web-ip[count.index].name
    resource_group_name = azurerm_virtual_machine.web-vm[count.index].resource_group_name
    depends_on = [
        azurerm_virtual_machine.web-vm
    ]
    count = var.web_frontend_count
}

data "azurerm_public_ip" "db_ip" {
    name = azurerm_public_ip.public-db-ip[count.index].name
    resource_group_name = azurerm_virtual_machine.db-vm[count.index].resource_group_name
    depends_on = [
        azurerm_virtual_machine.db-vm
    ]
    count = var.db_backend_count
}

//start test

resource "azurerm_availability_set" "web-set" {
    name = "DaveTF-Web-Availability-Set"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    platform_fault_domain_count = 2
    platform_update_domain_count = 2
}

resource "azurerm_public_ip" "lb_ip" {
    name = "DaveTF-LB-PubIP"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    allocation_method = "Static"
}

resource "azurerm_lb" "lb" {
    name = "DaveTF-LB"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name

    frontend_ip_configuration {
      name = "Web"
      public_ip_address_id = azurerm_public_ip.lb_ip.id
    }
}

resource "azurerm_lb_backend_address_pool" "lb-backend-pool" {
    resource_group_name = azurerm_resource_group.rg.name
    loadbalancer_id = azurerm_lb.lb.id
    name = "DaveTF-Web-Backend-Pool"
}

resource "azurerm_network_interface" "lb-internal-nic" {
    name = "DaveTF-LB-Internal-NIC"
    location = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name

    ip_configuration {
      name = "DaveTF-LB-Internal-NIC-Config"
      subnet_id = azurerm_subnet.web-subnet.id
      private_ip_address_allocation = "Dynamic"
    }
}

resource "azurerm_network_interface_backend_address_pool_association" "web-backend-association" {
    network_interface_id = azurerm_network_interface.web-nic[count.index].id  //azurerm_network_interface.lb-internal-nic.id
    ip_configuration_name = "DaveTF-Web-NICConfig"
    backend_address_pool_id = azurerm_lb_backend_address_pool.lb-backend-pool.id
    count = var.web_frontend_count
}

resource "azurerm_lb_rule" "lb-web-in" {
    resource_group_name = azurerm_resource_group.rg.name
    loadbalancer_id = azurerm_lb.lb.id
    name = "Web-In"
    protocol = "Tcp"
    frontend_port = 80
    backend_port = 80
    frontend_ip_configuration_name = "Web"
    backend_address_pool_id = azurerm_lb_backend_address_pool.lb-backend-pool.id
    probe_id = azurerm_lb_probe.web-probe.id
    depends_on = [
      azurerm_lb_probe.web-probe
    ]
}

resource "azurerm_lb_probe" "web-probe" {
    resource_group_name = azurerm_resource_group.rg.name
    loadbalancer_id = azurerm_lb.lb.id
    name = "HTTP-Probe"
    port = 80
}

//end test

resource "local_file" "tf_ansible_inventory" {
    content = <<-DOC
    ---
    # Ansible inventory generated by terraform
    
    all:
      vars:
        ansible_user: ${var.admin_username}
      children:
        web_frontends:
          hosts:
          %{ for ip in data.azurerm_public_ip.web_ip ~}
  ${ip.ip_address}:
          %{ endfor ~}

        db_backends:
          hosts:
          %{ for ip in data.azurerm_public_ip.db_ip ~}
  ${ip.ip_address}:
          %{ endfor ~}
    DOC
    filename = "./tf_ansible_inventory"
}  


