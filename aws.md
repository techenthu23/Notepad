Here's a shell script that automates the process of listing AWS Service Catalog provisioned products, identifying their CloudFormation stacks, and fetching resource details:


---

Shell Script: get_servicecatalog_resources.sh

#!/bin/bash

# Exit on any error
set -e

# Optional: set AWS profile if needed
# export AWS_PROFILE=your-profile-name

echo "Fetching list of provisioned products..."
provisioned_products=$(aws servicecatalog list-provisioned-products --output json)

# Extract IDs and Names
product_count=$(echo "$provisioned_products" | jq '.ProvisionedProducts | length')

if [ "$product_count" -eq 0 ]; then
    echo "No provisioned products found."
    exit 0
fi

echo "$product_count provisioned products found."

for i in $(seq 0 $((product_count - 1))); do
    product_id=$(echo "$provisioned_products" | jq -r ".ProvisionedProducts[$i].Id")
    product_name=$(echo "$provisioned_products" | jq -r ".ProvisionedProducts[$i].Name")

    echo ""
    echo "Product: $product_name"
    echo "ID: $product_id"

    # Describe provisioned product
    product_detail=$(aws servicecatalog describe-provisioned-product --id "$product_id")
    physical_id=$(echo "$product_detail" | jq -r '.ProvisionedProductDetail.PhysicalId')

    echo "Associated CloudFormation Stack: $physical_id"

    if [ "$physical_id" == "null" ]; then
        echo "No CloudFormation stack associated."
        continue
    fi

    # Create folder for each product
    mkdir -p "output/$product_name"

    echo "Fetching stack details for $physical_id..."

    # Describe the stack
    aws cloudformation describe-stacks --stack-name "$physical_id" \
        --output json > "output/$product_name/stack-details.json"

    # List stack resources
    aws cloudformation describe-stack-resources --stack-name "$physical_id" \
        --output json > "output/$product_name/stack-resources.json"

    echo "Details saved in output/$product_name/"
done

echo ""
echo "Done. All stack and resource details saved in the output/ folder."


---

Prerequisites

AWS CLI configured (aws configure)

jq installed (sudo apt install jq or brew install jq)



---

Run the Script

Save as get_servicecatalog_resources.sh, then:

chmod +x get_servicecatalog_resources.sh
./get_servicecatalog_resources.sh


---

Would you like me to enhance this to include tags or resource types from the stack as well?

