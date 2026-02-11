# Use official Qdrant image
FROM qdrant/qdrant:latest

# Metadata
LABEL maintainer="local-exploration"

# Expose ports
EXPOSE 6333  
EXPOSE 6334   

# Persistent storage mount point
VOLUME ["/qdrant/storage"]

# Start Qdrant
CMD ["./qdrant"]
